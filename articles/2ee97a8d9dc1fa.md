---
title: "gRPC-Go+CloudTraceで分散トレーシング"
emoji: "🐨"
type: "tech"
topics: ["golang", "grpc", "opentelemetry", "cloudtrace", "trace"]
published: true
---

# はじめに
## 以前
以前に[Akka gRPC（Scala）+CloudTraceで分散トレーシング](https://zenn.dev/lightstaff/articles/e392deef27ef96)なんて記事を書きましたが、Akka gRPCだけじゃなくてGo言語（gRPC-Go）でもサービスが動いてたんで追加してみました。

※CloudTraceおよびOpenTelemetryについての説明は割愛（上記記事または各ドキュメントを参照してください）。

※GCPドキュメント上ではGoでのOpenTelemetryの利用はβ扱いになっています。使用ライブラリである[OpenTelemetry-Go](https://github.com/open-telemetry/opentelemetry-go)を見ると「Traces」はStableになっているのでトレースのみなら問題なさそうです。

# Goで実装
## 導入は楽だよ
Akka gRPCの時もそうでしたが分散トレーシングを考えない場合、[GoとOpenTelemetry](https://cloud.google.com/trace/docs/setup/go-ot)を参考にすれば特に苦も無く実装できます。

ただ、Akka gRPC同様に分散トレーシングしたい場合はコンテキストをプロパゲーション（伝搬）する必要があります。

Akka gRPCではJavaとの兼ね合いでコンテキストのパーサーであるプロパーゲーター（伝搬屋）を自力実装したりInterceptorが無い等、結構面倒なことになってましたね。

それに比べてgRPC-Goは、[W3C準拠](https://www.w3.org/TR/trace-context/)のプロパゲーターがそのまま使えるし、Interceptorもあるしで楽勝です。

## 前準備
[GoとOpenTelemetry](https://cloud.google.com/trace/docs/setup/go-ot)の通り、必要なライブラリをインポートし、トレーサーを初期化します。

そこにプロパゲーターを指定する設定を追加します。←**必須**

```go
// https://cloud.google.com/trace/docs/setup/go-otを一部改変
package main

import (
        "context"
        "log"
        "os"

        texporter "github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/trace"
        "go.opentelemetry.io/otel"
        sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

func main() {
        ctx := context.Background()
        projectID := os.Getenv("GOOGLE_CLOUD_PROJECT")
        exporter, err := texporter.New(texporter.WithProjectID(projectID))
        if err != nil {
                log.Fatalf("texporter.NewExporter: %v", err)
        }

        tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(exporter))
        defer tp.ForceFlush(ctx) // flushes any pending spans
        otel.SetTracerProvider(tp)

        // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
        otel.SetTextMapPropagator(propagation.TraceContext{})
        // ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
	
        tracer := otel.GetTracerProvider().Tracer("example.com/trace")
        err = func(ctx context.Context) error {
                ctx, span := tracer.Start(ctx, "foo")
                defer span.End()

                // Do some work.

                return nil
        }(ctx)
}
```

## Interceptorを実装
gRPC-GoにはInterceptorがあるのでそこにトレーサーを組み込みましょう。

その前にキャリアを更新・削除用にプロパゲーターに引き渡す`TextMapCarrier`インタフェースに従った定義が必要になります。今回はgRPC-GoなのでMetadata（`google.golang.org/grpc/metadata.MD`）を拡張します。

```go
package tracing

// importは省略

type MetadataCarrier metadata.MD

func (mc MetadataCarrier) Get(key string) string {
	md := metadata.MD(mc)

	v := md.Get(key)
	if len(v) == 0 {
		return ""
	}

	return v[0]
}

func (mc MetadataCarrier) Set(key string, value string) {
	md := metadata.MD(mc)

	md.Append(key, value)
}

func (mc MetadataCarrier) Keys() []string {
	md := metadata.MD(mc)

	keys := make([]string, 0, len(md))

	for k := range md {
		keys = append(keys, k)
	}

	return keys
}
```

上記を利用してInterceptorを定義します。

Interceptorは全サービスが通ってしまうので素朴なフィルタも考えてみました。

```go
package tracing

// importは省略

var (
	IGNORE_METHOD = map[string]interface{}{
		"/grpc.health.v1.Health/Check": nil,
		"/grpc.health.v1.Health/Watch": nil,
	}
)

func UnaryServerInterceptor() grpc.UnaryServerInterceptor {
	return func(ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (interface{}, error) {
		if _, ignore := IGNORE_METHOD[info.FullMethod]; ignore {
			return handler(ctx, req)
		}

		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return handler(ctx, req)
		}

		copyMD := md.Copy()

		ctx = otel.GetTextMapPropagator().Extract(ctx, MetadataCarrier(copyMD))

		ctx, span := otel.GetTracerProvider().Tracer(TRACER_NAME).Start(
			ctx,
			info.FullMethod,
			trace.WithSpanKind(trace.SpanKindServer),
		)
		defer span.End()

		res, err := handler(ctx, req)
		if err != nil {
			span.SetStatus(codes.Error, err.Error())
		}

		return res, err
	}
}

func StreamServerInterceptor() grpc.StreamServerInterceptor {
	return func(
		srv interface{},
		ss grpc.ServerStream,
		info *grpc.StreamServerInfo,
		handler grpc.StreamHandler,
	) error {
		if _, ignore := IGNORE_METHOD[info.FullMethod]; ignore {
			return handler(srv, ss)
		}

		ctx := ss.Context()

		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return handler(srv, ss)
		}

		copyMD := md.Copy()

		ctx = otel.GetTextMapPropagator().Extract(ctx, MetadataCarrier(copyMD))

		tracer := otel.GetTracerProvider().Tracer(TRACER_NAME)

		_, span := tracer.Start(
			ctx,
			info.FullMethod,
			trace.WithSpanKind(trace.SpanKindServer),
		)
		defer span.End()

		err := handler(srv, ss)
		if err != nil {
			span.SetStatus(codes.Error, err.Error())
		}

		return err
	}
}
```

あとはこれらをサーバーの初期化時に噛ませるだけです。

```go
// 初期化部分のみ抜粋
grpcServer := grpc.NewServer(
	grpc.StreamInterceptor(tracing.StreamServerInterceptor()),
	grpc.UnaryInterceptor(tracing.UnaryServerInterceptor()),
)
```

# 終わりに
## まとめ
- gRPC-Go+CloudTraceは楽。