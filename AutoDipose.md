# AutoDispose の動作

```text
riverpodはproviderにリスナーがいるかどうかを追跡し、providerがリスナーを持たない場合、
stateが破棄される。
```

```text
@riverpodアノテーションによるコード生成を用いた場合は、デフォルトでautoDisposeが指定される
```

# リスナが全て削除されてから、一定時間 Provider の破棄を遅らせる

ファイル構成例 (lib/extensions/provider_extensions.dart):

```dart
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';

/// Refに拡張メソッドを追加して、リスナーが全て削除された後にプロバイダーを一定時間保持します。
extension DelayedDisposeExtension on Ref {
  /// リスナーが全て削除された後、指定された[duration]だけプロバイダーを保持します。
  void keepAliveAfterDispose(Duration duration) {
    // keepAliveリンクを取得してプロバイダーの状態を保持
    final link = keepAlive();
    Timer? timer;

    // リスナーが全て削除された際に呼び出されるコールバック
    ref.onCancel(() {
      // タイマーを設定して、[duration]後にkeepAliveリンクを閉じる
      timer = Timer(duration, () {
        link.close();
      });
    });

    // 新たなリスナーが追加された際に呼び出されるコールバック
    ref.onResume(() {
      // タイマーが存在する場合はキャンセル
      timer?.cancel();
      timer = null;
    });

    // プロバイダーが破棄される際にタイマーをキャンセル
    ref.onDispose(() {
      timer?.cancel();
    });
  }
}

```

ファイル構成例 (lib/providers/delayed_dispose_provider.dart):

```dart
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

/// autoDisposeプロバイダーを定義し、リスナーが全て削除された後に5分間状態を保持します。
final delayedDisposeProvider = FutureProvider.autoDispose<Object>((ref) async {
  /// リスナーが全て削除された後、5分間状態を保持します
  ref.keepAliveAfterDispose(const Duration(minutes: 5));

  // 実際の非同期操作（例: HTTPリクエスト）
  final response = await http.get(Uri.https('example.com'));
  return response.body;
});

```

参考: https://riverpod.dev/docs/essentials/auto_dispose#example-keeping-state-alive-for-a-specific-amount-of-time
