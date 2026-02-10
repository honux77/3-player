# MP3 백그라운드 재생 구현 계획

## 목표
화면이 꺼져도 음악이 계속 재생되도록 VGM → MP3 변환 방식으로 전환

## 현재 문제점
- VGM은 ScriptProcessor에서 실시간 에뮬레이션으로 오디오 생성
- 화면이 꺼지면 JavaScript 실행 중단 → 오디오 생성 멈춤
- HTMLAudioElement는 브라우저가 백그라운드에서도 재생 지원

## 구현 방식

### Phase 1: VGM → MP3 변환 파이프라인

#### 1-1. 변환 도구 설치
- `vgmplay` CLI 도구 사용 (VGM → WAV)
- `ffmpeg` 사용 (WAV → MP3)

#### 1-2. 변환 스크립트 작성
```
frontend/scripts/convert-vgm-to-mp3.js
```
- 각 ZIP 파일에서 VGM/VGZ 추출
- vgmplay로 WAV 변환
- ffmpeg로 MP3 변환 (128kbps 또는 192kbps)
- MP3 파일을 새 ZIP으로 재압축 또는 폴더 구조로 저장

#### 1-3. 출력 구조
```
frontend/public/music-mp3/
  ├── Daikoukai_Jidai_(Sharp_X68000)/
  │   ├── 01_Waves.mp3
  │   ├── 02_Stormy_Voyage.mp3
  │   └── ...
  └── ...
```

### Phase 2: 프론트엔드 수정

#### 2-1. useVGMPlayer.js 수정
- ScriptProcessor 대신 HTMLAudioElement 사용
- Audio 엘리먼트를 Web Audio API의 AnalyserNode에 연결 (시각화 유지)

```javascript
const audio = new Audio(mp3Url);
const source = audioContext.createMediaElementSource(audio);
source.connect(analyser);
analyser.connect(audioContext.destination);
audio.play();
```

#### 2-2. 트랙 로딩 방식 변경
- ZIP 압축 해제 불필요
- manifest.json에 MP3 경로 추가
- 직접 MP3 URL로 재생

#### 2-3. 기존 기능 유지
- [x] 주파수 시각화 (AnalyserNode 연결)
- [x] Media Session API (잠금화면 컨트롤)
- [x] 자동 다음 트랙 재생
- [x] 이전/다음 트랙 버튼

### Phase 3: manifest.json 업데이트

#### 3-1. 새 필드 추가
```json
{
  "id": "Daikoukai_Jidai__Sharp_X68000_",
  "title": "Daikoukai Jidai",
  "tracks": [
    {
      "filename": "01 Waves.vgz",
      "mp3": "music-mp3/Daikoukai_Jidai_(Sharp_X68000)/01_Waves.mp3",
      "name": "Waves",
      "duration": 125
    }
  ]
}
```

### Phase 4: 빌드 프로세스 통합

#### 4-1. npm 스크립트 추가
```json
{
  "scripts": {
    "convert": "node scripts/convert-vgm-to-mp3.js",
    "build": "npm run convert && vite build"
  }
}
```

## 작업 순서

1. [ ] vgmplay, ffmpeg 설치 및 테스트
2. [ ] 변환 스크립트 작성 (1개 게임으로 테스트)
3. [ ] useVGMPlayer.js를 useAudioPlayer.js로 리팩토링
4. [ ] HTMLAudioElement + AnalyserNode 연결 구현
5. [ ] 전체 게임 MP3 변환
6. [ ] manifest.json 업데이트 스크립트 작성
7. [ ] 테스트 (데스크톱 + 모바일)
8. [ ] 배포

## 고려사항

### 장점
- 화면 꺼져도 백그라운드 재생 가능
- 모바일 브라우저 호환성 향상
- CPU 사용량 감소 (에뮬레이션 불필요)

### 단점
- 저장 용량 증가 (VGM < MP3)
- 빌드 시간 증가
- 새 게임 추가 시 변환 필요

### 용량 예상
- VGM ZIP: ~1-5MB per game
- MP3 (128kbps): ~2-10MB per game
- 총 약 2배 용량 예상

## 대안 (향후 고려)

### AudioWorklet 방식
- ScriptProcessor 대신 AudioWorklet 사용
- 일부 브라우저에서 백그라운드 실행 개선
- 단, 여전히 JS 실행 필요하므로 완전한 해결책 아님

### 서버 사이드 스트리밍
- 서버에서 실시간 VGM → Audio 스트리밍
- 인프라 비용 발생
