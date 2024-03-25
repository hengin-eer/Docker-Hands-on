# Docker-Hands-on

Docker ハンズオン用のリポジトリです！

## 学習メモ
Go Serverの準備は大変なので、とりあえずNext.jsの環境構築とDocker Composeでの軌道を目標とした。
ルートディレクトリで`docker-compose up -d`を実行すると、以下のようなログが表示された。

```bash
2024-03-25 23:19:11 next-front-1  | 
2024-03-25 23:19:11 next-front-1  | > app@0.1.0 dev
2024-03-25 23:19:11 next-front-1  | > next dev
2024-03-25 23:19:11 next-front-1  | 
2024-03-25 23:19:22 next-front-1  |    ▲ Next.js 14.1.4
2024-03-25 23:19:22 next-front-1  |    - Local:        http://localhost:3000
2024-03-25 23:19:22 next-front-1  | 
2024-03-25 23:19:22 next-front-1  | Attention: Next.js now collects completely anonymous telemetry regarding usage.
2024-03-25 23:19:22 next-front-1  | This information is used to shape Next.js' roadmap and prioritize features.
2024-03-25 23:19:22 next-front-1  | You can learn more, including how to opt-out if you'd not like to participate in this anonymous program, by visiting the following URL:
2024-03-25 23:19:22 next-front-1  | https://nextjs.org/telemetry
2024-03-25 23:19:22 next-front-1  | 
2024-03-25 23:19:39 next-front-1  |  ✓ Ready in 18.2s
2024-03-25 23:19:40 next-front-1  |  ○ Compiling / ...
2024-03-25 23:19:56 next-front-1  |  ✓ Compiled / in 17.5s (523 modules)
2024-03-25 23:19:59 next-front-1  |  ○ Compiling /favicon.ico ...
2024-03-25 23:20:05 next-front-1  |  ✓ Compiled /favicon.ico in 6.1s (521 modules)
```

`docker-compose stop`をターミナルで実行すると、開発サーバーが停止したことを確認できた。
（ブラウザのページにアクセスできなくなった。）
Docker Desktopから確認すると、停止状態になっているのみであり、再開ボタンを押すと再び起動した。
コマンドラインから操作しても良いけど、若干のラグがあったため、ソフト側から操作したほうがログの確認もスムーズだし良いかも。

---

ルート直下 Dockerfile

```Dockerfile
FROM node:20.11.1

WORKDIR /app
```

ルート直下 compose.yaml

```yaml
services:
  next-front:
    build:
      context: .
    tty: true
    volumes:
      - ./next-front:/app
    environment:
      - WATCHPACK_POLLING=true
    command: sh -c "npm run dev"
    ports:
      - "3000:3000"
```

## 〜コマンドについて〜

サービス app が起動してコンテナが作られ、実行が終わり次第コンテナは削除される。→ —rm

コンテナ内でシェルを実行 → sh

ファイル名の代わりに実行するコマンドの文字列であることを伝える → -c

カレントディレクトリに作成 → .

src ディレクトリを作成しない → —no-src-dir

typescript を入れる → —typescript

```shell
docker compose run --rm next-front sh -c 'npx create-next-app . --no-src-dir --typescript'
```

ルート直下で以下のコマンドを実行する

```shell
mkdir go-server
cd go-server
go mod init go-server
touch main.go
```

main.go の中身

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, World!")
	})

	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

```shell
go run main.go
curl localhost:8080
```

/go-server/Dockerfile を作る

```Dockerfile
FROM golang:1.22.1-bookworm
WORKDIR /app
RUN go install github.com/cosmtrek/air@latest
CMD ["air"]
```

compose.yaml に以下追記

```yaml
  go-server:
    build:
      context: ./go-server
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    tty: true
    volumes:
      - ./go-server:/app
```

/go-server/.air.toml の追加

```toml
# Config file for [Air](https://github.com/cosmtrek/air) in TOML format

# Working directory

# . or absolute path, please note that the directories following must be under root.

root = "."
tmp_dir = "tmp"

[build]

# Array of commands to run before each build

pre_cmd = ["echo 'hello air' > pre_cmd.txt"]

# Just plain old shell command. You could use `make` as well.

cmd = "go build -o ./tmp/main ."

# Array of commands to run after ^C

post_cmd = ["echo 'hello air' > post_cmd.txt"]

# Binary file yields from `cmd`.

bin = "tmp/main"

# Customize binary, can setup environment variables when run your app.

full_bin = "APP_ENV=dev APP_USER=air ./tmp/main"

# Watch these filename extensions.

include_ext = ["go", "tpl", "tmpl", "html"]

# Ignore these filename extensions or directories.

exclude_dir = ["assets", "tmp", "vendor", "frontend/node_modules"]

# Watch these directories if you specified.

include_dir = []

# Watch these files.

include_file = []

# Exclude files.

exclude_file = []

# Exclude specific regular expressions.

exclude_regex = ["_test\\.go"]

# Exclude unchanged files.

exclude_unchanged = true

# Follow symlink for directories

follow_symlink = true

# This log file places in your tmp_dir.

log = "air.log"

# Poll files for changes instead of using fsnotify.

poll = false

# Poll interval (defaults to the minimum interval of 500ms).

poll_interval = 500 # ms

# It's not necessary to trigger build each time file changes if it's too frequent.

delay = 0 # ms

# Stop running old binary when build errors occur.

stop_on_error = true

# Send Interrupt signal before killing process (windows does not support this feature)

send_interrupt = false

# Delay after sending Interrupt signal

kill_delay = 500 # nanosecond

# Rerun binary or not

rerun = false

# Delay after each executions

rerun_delay = 500

# Add additional arguments when running binary (bin/full_bin). Will run './tmp/main hello world'.

args_bin = ["hello", "world"]

[log]

# Show log time

time = false

# Only show main log (silences watcher, build, runner)

main_only = false

[color]

# Customize each part's color. If no color found, use the raw app log.

main = "magenta"
watcher = "cyan"
build = "yellow"
runner = "green"

[misc]

# Delete tmp directory on exit

clean_on_exit = true

[screen]
clear_on_rebuild = true
keep_scroll = true
```

compose.yaml がある場所で以下のコマンド実行

```
docker-compose up -d go-server
```
