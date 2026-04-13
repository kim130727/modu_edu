
그리고 기존 README도 약간 수정하는 것이 좋습니다.  
특히 `modu_edu`가 지금은 영어·국어·사회 예시 저장소라 해도, 방금 주신 규약은 수학 쪽과 훨씬 더 직접 연결되어 있습니다.  
그래서 README 문장을 이렇게 바꾸면 더 정확합니다.

기존:
- 과목별 학습 콘텐츠를 semantic JSON 형태로 구조화하기 위한 실험 저장소

수정 추천:
- 교육용 문제 데이터를 canonical semantic JSON / layout JSON 구조로 정리하기 위한 실험 저장소
- 구조뿐 아니라 필드 순서까지 고정한 표준 출력 규약을 지향
- semantic, render, answer, layout, diff를 일관된 계약(contract)으로 관리

한마디로 말하면,  
지금 주신 것은 단순 보조 설정이 아니라 README에서 강조해야 할 핵심 철학입니다.

원하시면 제가 지금 바로 이 field order까지 반영해서  
README 전체를 새로 완성본으로 다시 써드리겠습니다.


# 작업 폴더 기준: C:\projects\modu_edu

# 1) 청크 MP3 합치기용 목록 생성
@"
file 'john_3_16_web_ch1.mp3'
file 'john_3_16_web_ch2.mp3'
file 'john_3_16_web_ch3.mp3'
file 'john_3_16_web_ch4.mp3'
file 'john_3_16_web_ch5.mp3'
"@ | Set-Content -Encoding utf8 output/chunks/concat.txt

# 2) 하나의 음성 파일로 합치기
ffmpeg -y -f concat -safe 0 -i output/chunks/concat.txt -c copy output/narration.mp3

# 3) 실제 MP3 길이 기준으로 SRT 자동 생성
@'
import subprocess
from pathlib import Path

chunks = [
    ("john_3_16_web_ch1.mp3", "For God so loved the world"),
    ("john_3_16_web_ch2.mp3", "that he gave his one and only Son"),
    ("john_3_16_web_ch3.mp3", "that whoever believes in him"),
    ("john_3_16_web_ch4.mp3", "should not perish"),
    ("john_3_16_web_ch5.mp3", "but have eternal life"),
]

def dur(path):
    out = subprocess.check_output([
        "ffprobe","-v","error","-show_entries","format=duration",
        "-of","default=noprint_wrappers=1:nokey=1", str(path)
    ], text=True).strip()
    return float(out)

def ts(sec):
    h = int(sec // 3600)
    m = int((sec % 3600) // 60)
    s = int(sec % 60)
    ms = int(round((sec - int(sec)) * 1000))
    if ms == 1000:
        s += 1
        ms = 0
    return f"{h:02}:{m:02}:{s:02},{ms:03}"

base = Path("output/chunks")
cur = 0.0
lines = []
for i, (fn, text) in enumerate(chunks, 1):
    d = dur(base / fn)
    start = cur
    end = cur + d
    lines += [str(i), f"{ts(start)} --> {ts(end)}", text, ""]
    cur = end

Path("output/subtitles.srt").write_text("\n".join(lines), encoding="utf-8")
print("created: output/subtitles.srt")
'@ | .venv\Scripts\python.exe -

# 4) 흰 배경 + 검은 자막 + 음성 MP4 생성
# (폰트 파일 경로는 필요시 변경)
ffmpeg -y -f lavfi -i "color=c=white:s=1280x720:d=600" -i output/narration.mp3 `
  -vf "subtitles=output/subtitles.srt:force_style='FontName=Arial,FontSize=52,PrimaryColour=&H00000000,Outline=0,Shadow=0,Alignment=2,MarginV=80'" `
  -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 192k -shortest output/john_3_16_white_subs.mp4
