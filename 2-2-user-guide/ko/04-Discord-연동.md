# Discord 연동

## 빠른 설정
1. 개발자 포털에서 Discord 봇 생성
2. **Message Content Intent** + **Server Members Intent** 활성화
3. 토큰 설정: `DISCORD_BOT_TOKEN=...` 또는 설정 파일
4. 메시지 권한으로 봇을 서버에 초대
5. 게이트웨이 시작

## 최소 설정

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN"
    }
  }
}
```

## 세션 키
- DM: `agent:<agentId>:main` (공유 메인 세션)
- 길드: `agent:<agentId>:discord:channel:<channelId>`

## DM 접근 제어

```json5
{
  channels: {
    discord: {
      dm: {
        enabled: true,
        policy: "pairing",  // pairing | allowlist | open | disabled
        allowFrom: ["123456789012345678", "username"]
      }
    }
  }
}
```

## 길드 설정

```json5
{
  channels: {
    discord: {
      guilds: {
        "GUILD_ID": {
          requireMention: true,
          users: ["USER_ID"],
          channels: {
            help: { 
              allow: true, 
              requireMention: true,
              skills: ["docs"],
              systemPrompt: "짧게 답변해."
            }
          }
        }
      }
    }
  }
}
```

## 멘션 게이팅
- `requireMention: true` (기본값): @멘션 시에만 응답
- 텍스트 기반 트리거는 `mentionPatterns` 사용

## 도구 액션

| 액션 | 기본값 | 설명 |
|------|--------|------|
| reactions | 활성화 | 리액션 추가/목록 |
| messages | 활성화 | 읽기/전송/편집/삭제 |
| threads | 활성화 | 생성/목록/답장 |
| pins | 활성화 | 고정/고정 해제/목록 |
| moderation | 비활성화 | 타임아웃/추방/차단 |

## 답장 스레딩
- 기본값: 꺼짐
- 활성화: `channels.discord.replyToMode: "first"` 또는 `"all"`
- `[[reply_to_current]]` 또는 `[[reply_to:<id>]]` 태그 사용

## 제한
- 텍스트 청크: 2000자 (설정 가능)
- 메시지당 최대 줄: 17
- 미디어: 기본 8 MB

## 문제 해결
- "Used disallowed intents": Message Content Intent 활성화
- 응답 없음: 봇 권한 + 허용 목록 확인
- `openclaw doctor`와 `openclaw channels status --probe` 실행
