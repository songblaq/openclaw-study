# WhatsApp 연동

## 빠른 설정
1. **별도 전화번호** 사용 (권장)
2. `~/.openclaw/openclaw.json`에 설정
3. `openclaw channels login` → QR 스캔
4. 게이트웨이 시작

## 최소 설정

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"]
    }
  }
}
```

## 전화번호 옵션

### 전용 번호 (권장)
- 여분 안드로이드폰 + eSIM
- 같은 기기에서 WhatsApp Business 사용 가능

### 개인 번호 (대안)
- 셀프챗 모드 활성화
- 자신에게 메시지로 테스트

```json5
{
  channels: {
    whatsapp: {
      selfChatMode: true,
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"]
    }
  }
}
```

## 로그인/로그아웃

```bash
# 로그인 (QR 스캔)
openclaw channels login
openclaw channels login --account <id>  # 멀티 계정

# 로그아웃
openclaw channels logout
```

자격증명 위치: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`

## DM 정책
| 정책 | 동작 |
|------|------|
| `pairing` (기본값) | 모르는 발신자에게 코드, 승인 필요 |
| `allowlist` | `allowFrom` 목록만 허용 |
| `open` | 모두 허용 (`["*"]` 필요) |
| `disabled` | 모든 DM 무시 |

## 그룹 설정

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groups: {
        "*": { requireMention: true }
      }
    }
  },
  agents: {
    list: [{
      id: "main",
      groupChat: { mentionPatterns: ["@openclaw", "응답해"] }
    }]
  }
}
```

## 멀티 계정

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {}
      }
    }
  }
}
```

## 자동 확인 리액션

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions"  // always | mentions | never
      }
    }
  }
}
```

## 제한
- 텍스트 청크: 4000자
- 인바운드 미디어: 50 MB
- 아웃바운드 미디어: 5 MB (자동 최적화)

## 문제 해결

**연결 안 됨**
```bash
openclaw channels login  # QR 다시 스캔
```

**연결됐지만 끊김**
```bash
openclaw doctor  # 또는 게이트웨이 재시작
```

**Bun 런타임 문제**
- WhatsApp (Baileys)은 Bun에서 불안정
- 게이트웨이는 **Node** 사용 권장
