---
title: CronJob
content_template: templates/concept
weight: 80
---

{{% capture overview %}}

_Cron Job_ は、以下のような時間ベースの[Job](/ja/docs/concepts/workloads/controllers/jobs-run-to-completion/)を作成します。

1つのCronJobオブジェクトは _crontab_ (cron table)ファイルの一行に似ています。Jobを[Cron](https://en.wikipedia.org/wiki/Cron)フォーマットで書かれたスケジュールで定期的に実行します。
{{< note >}}
**メモ:** すべての **CronJob** `schedule:` 時刻はUTCで記述されます。
{{< /note >}}

Cron Jobを作成し利用するための説明と、スペックファイルの例は[Cron Jobで自動化されたタスクを実行する](/ja/docs/tasks/job/automated-tasks-with-cron-jobs)を参照してください。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## Cron Jobの制限

Cron Jobはそのスケジュールの実行時間あたり"約"1回、Jobオブジェクトを作成します。"約"と言ったのは、特定の状況ではJobが2つ作られたり、全く作られなかったりする可能性があるためです。この状況をまれなものにしようとしていますが、完全に回避できてはいません。したがって、Jobは _冪等_ であるべきです。

`startingDeadlineSeconds`が大きな値に設定されている、もしくは設定されておらず (これがデフォルト)、`concurrencyPolicy`が`Allow`に設定されていれば、Jobは常に少なくとも1回は実行されます。

すべてのCronJobに対して、CronJobコントローラは最後にスケジュールされてから現在までの期間でミスしたスケジュールの回数をチェックします。100回を超えてミスしたスケジュールがあれば、そのJobは起動せずログにエラーを記録します。

````
Cannot determine if job needs to be started. Too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.
````

`startingDeadlineSeconds`フィールドが設定されている (`nil`でない) と、コントローラは`startingDeadlineSeconds`の値から現在までで発生したミスしたJobの数をカウントすることに注意してください。例えば、`startingDeadlineSeconds`が`200`であれば、コントローラは200秒前までで発生したミスしたJobの数をカウントします。

CronJobはスケジュールされた時間での作成に失敗するとミスとカウントします。例えば、`concurrencyPolicy`が`Forbid`に設定されていて、CronJobが前のスケジュールのものがまだ動作している時にスケジュールしようとすると、ミスとカウントされます。

例えば、あるCronJobが`08:30:00`丁度に開始するよう設定されていて、その`startingDeadlineSeconds`が10に設定されていて、CronJobコントローラが`08:29:00`から`08:42:00`までダウンしていたとすると、Jobは起動されません。全く起動されないよりも遅れてでも起動したほうがいいのなら、長めの`startingDeadlineSeconds`を設定しておきます。

CronJobはマッチしたスケジュールにJobを起動することにのみ責任を負い、そのJobはPodの管理に責任を負います。

{{% /capture %}}
