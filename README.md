# Streamlit 수정 → PR 머지 → WSL 자동 배포

**교사 저장소에는 `app.py`와 이 안내서만 있다.** 학생은 clone 후 자기 저장소에서 Actions 파일을 만든다.

준비: Windows Git, GitHub 계정, WSL Ubuntu, Python 3.11 이상과 venv.
명령은 **CMD / WSL** 표시를 구분해서 실행한다.

## 1. 예제 clone — CMD

```cmd
git clone https://github.com/futureStudyRepo/workspace_git.git streamlit-demo
cd streamlit-demo
```

## 2. 내 GitHub 저장소 연결 — GitHub + CMD

GitHub에서 **내 계정 / 이름 `streamlit-demo` / Private / 초기 파일 없음**으로 빈 저장소를 만든다.
`YOUR_ID`를 본인 아이디로 바꾼다.

```cmd
git remote set-url origin https://github.com/YOUR_ID/streamlit-demo.git
```

## 3. Actions 파일 만들고 첫 push — CMD

```cmd
mkdir .github\workflows
notepad .github\workflows\deploy.yml
```

아래 내용을 저장한다. **label이나 경로를 따로 수정하지 않는다.**

```yaml
name: Streamlit 배포

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency: streamlit-deploy

jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: [self-hosted, Linux]
    steps:
      - uses: actions/checkout@v4
      - name: 문법 검사
        run: python3 -m py_compile app.py
      - name: WSL에 배포
        run: |
          mkdir -p "$HOME/streamlit-demo"
          cp app.py "$HOME/streamlit-demo/app.py.tmp"
          mv "$HOME/streamlit-demo/app.py.tmp" "$HOME/streamlit-demo/app.py"
```

```cmd
git add .github/workflows/deploy.yml
git commit -m "Add deployment workflow"
git push -u origin main
```

로그인이 나오면 본인 계정으로 인증한다. 이때 Actions가 Queued인 것은 runner를 아직 실행하지 않았기 때문이다.

## 4. 내 WSL runner 등록 — WSL 창 ①

내 저장소의 **Settings → Actions → Runners → New self-hosted runner → Linux / x64**를 연다.
WSL에서 `cd ~`로 홈에 이동한 뒤 GitHub 화면의 **Download 명령**을 실행한다.
압축을 푼 runner 폴더에서 등록한다.

```bash
./config.sh
```

질문에 **내 저장소 URL**, GitHub 화면의 **등록 토큰**을 입력한다. 나머지는 Enter로 기본값을 쓴다. 토큰을 코드·문서에 저장하지 않는다.

```bash
./run.sh
```

`Listening for Jobs`가 나오면 창을 열어 둔다. GitHub **Actions의 문법 검사·배포가 성공**할 때까지 기다린다.

## 5. Streamlit 켜기 — 별도 WSL 창 ②

runner와 **같은 WSL 사용자**로 실행한다. 가상환경 생성과 설치는 처음 한 번만 한다.

```bash
python3 -m venv ~/streamlit-env
source ~/streamlit-env/bin/activate
pip install streamlit
cd ~/streamlit-demo
streamlit run app.py --server.address 127.0.0.1 --server.runOnSave true --server.headless true
```

Windows 브라우저에서 **[http://localhost:8501](http://localhost:8501)** 을 열어 **버전 1**을 확인한다. 서버 창을 열어 둔다.

## 6. 수정 → PR → main 머지 — 처음 CMD 창

```cmd
git switch -c codex/version2
notepad app.py
```

`st.header("버전 1")`을 **`st.header("버전 2")`** 로 바꾸고 저장한다.

```cmd
git add app.py
git commit -m "Change page to version 2"
git push -u origin codex/version2
```

내 저장소에서 **Pull requests → New pull request → base: main / compare: codex/version2 → Create → Merge** 순서로 진행한다.
**Actions가 성공하면 브라우저를 새로고침한다. 버전 2가 보이면 완료다.**

---

- CI는 Python 문법 검사, CD는 WSL 배포 폴더로 복사다. 문법 검사는 앱의 모든 동작을 검증하지는 않는다.
- PR 생성만으로는 실행되지 않는다. main 머지·직접 push·main 수동 실행에서 배포한다.
- 문법 검사가 실패하면 이전 배포 파일을 유지한다.
- PC·WSL·runner·Streamlit 서버를 켜 둔다. 종료는 각 실행 창에서 `Ctrl+C`다. 자동 시작 서비스는 사용하지 않는다.
- 다음 실습에는 runner 폴더의 `./run.sh`, WSL 창 ②의 가상환경 활성화와 Streamlit 실행을 다시 수행한다.
- venv가 없으면 `sudo apt install python3-venv`, runner가 ICU 오류를 내면 runner 폴더에서 `sudo ./bin/installdependencies.sh`로 준비한다.
- Git 이름·이메일 설정이 없다면 첫 commit 전에 본인 정보로 설정한다. 기존 로컬 폴더와 겹치면 다른 빈 위치에서 clone한다.
- 개인 PC runner는 **본인의 비공개 저장소**에 등록한다. 이 workflow에는 외부 PR 실행 이벤트가 없다.

참고: [Streamlit 설치](https://docs.streamlit.io/get-started/installation/command-line), [파일 변경 시 재실행 설정](https://docs.streamlit.io/develop/api-reference/configuration/config.toml).
