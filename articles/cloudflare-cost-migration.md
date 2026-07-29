Title
Vercel・Supabase・Resendが高い個人開発者が、Cloudflare中心の構成を検討した話

Emoji
☁️

Type
tech

Topics
cloudflare,vercel,supabase,個人開発,コスト

Published
true

Vercel + Supabase + Resend の料金がじわじわ気になってきた個人開発者向けの記事です。

はじめに
個人開発や受託で Next.js を使っていると、いつの間にか Vercel + Supabase + Resend が定番構成になっていました。開発体験は最高なのですが、案件やプロジェクトが増えるにつれて「このまま各サービスの料金が積み上がると、固定費がじわじわ効いてくるな」と感じるように。

そこで一度立ち止まって、コストを下げる代替構成を調べ直しました。この記事はその比較と、たどり着いた「Cloudflare中心の構成」のメモです。同じように費用が気になり始めた人の参考になれば。

そもそも Cloudflare は何をしてくれるのか
最初に自分がつまずいたのがここでした。Cloudflare は元々「サイトを速く・安全にする会社」で、そこに ホスティング機能（Pages / Workers） が加わった、と捉えると腑に落ちます。

CDN / 高速化 … 世界中に分散配信
DNS … ドメインとサーバーを繋ぐ
セキュリティ … DDoS防御・WAF・SSL自動付与
ホスティング（Pages / Workers） … アプリを動かす場所
ストレージ系（R2 / D1 / KV） … ファイル・DB・簡易保存
注意点として、Cloudflare はメールの「送信」は担当しません（できるのは受信の転送＝Email Routing のみ）。ここは後述の通り別サービスと組み合わせます。

置き換えの対応表
今の構成を、役割ごとに分解して代替候補を並べるとこうなりました。

役割	今まで	代替候補
ホスティング	Vercel	Cloudflare Pages / Workers（帯域無料が大きい）
データベース	Supabase (Postgres)	Neon（Postgres）/ Cloudflare D1（SQLite）
ストレージ	Supabase Storage	Cloudflare R2（下り転送無料）
認証	Supabase Auth	Better Auth / Clerk / Auth.js
メール送信	Resend	Amazon SES / Brevo
ポイントは「Supabaseを丸ごと1つで捨てる」のではなく、役割ごとに分解して安い部品に置き換えるという発想です。

なぜ Cloudflare だと帯域課金が消えるのか
Vercel はスケールすると帯域課金が効いてきますが、Cloudflare は帯域が無料で、有料プラン（Workers Paid）も月$5から。トラフィックが増えるほど差が開きます。

一方で、Next.js 特有の最適化（ISR・画像最適化など）は Vercel が一番スムーズなのも事実。なので自分の結論は「コスト重視の案件は Cloudflare、Next.js の機能をフルで使うこだわり案件は Vercel」という使い分けにしました。

メール送信は別で用意する
前述の通り Cloudflare はメール送信をしないので、ここだけ別サービスが必要です。

とにかく安く・大量に → Amazon SES（1,000通あたり$0.10、月額固定なし）
無料枠・使いやすさ → Brevo（無料で1日300通）
Resend の開発体験は良かったのですが、上位プランの値上げが乗り換えを考えたきっかけでした。用途が「アプリからの通知メール」中心なら SES で十分だと判断しています。

たどり着いた構成
最終的に、小〜中規模で、Next.js の画像最適化などを多用しないという前提で、コスト重視のベースはこの形に落ち着きました。

ホスティング：Cloudflare Pages / Workers
DB：Neon（Postgresを続けたい場合）or Cloudflare D1
ストレージ：Cloudflare R2
認証：Better Auth
メール送信：Amazon SES or Brevo
この規模感なら月数ドル、構成次第では実質無料の範囲で回せる見込みです。

おわりに
「Vercel + Supabase + Resend」は開発体験が優秀で、スピード重視の立ち上げには今でも最適解だと思います。ただ、運用が長くなりコストが気になってきたら、役割ごとに分解して安い部品に置き換えるという選択肢は持っておいて損はないな、というのが今日の学びでした。

実際に移行したら、その手順や詰まったポイントもまた記事にしようと思います。
