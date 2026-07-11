# asking-flow.md — Context flow ของ AI tutor (single endpoint)

> อัปเดต 2026-07-10 (Tier 0.D) — เอกสารนี้แทนที่เวอร์ชันเก่าที่อธิบาย legacy wiring (`/chat/ask`, `/chat/feedback`, `/chat/write-evaluate` ถูกลบไปแล้วใน Phase 0.A)
> เป้าหมาย: อ่านแล้ว **ทำนาย prompt ที่ Gemini เห็นได้เป๊ะๆ** ของทุก mode

ทุก AI surface บน client ยิงไปที่ **`POST /api/chat/tutor` ที่เดียว** ผ่าน bridge ตัวเดียว `client/src/components/editor/extensions/tutorApi.ts` (`callTutor`) — server เป็นเจ้าของ context ทั้งหมด (client ไม่ส่งเนื้อหาบทเรียนแล้ว)

## 1. ห้า surfaces → mode

| Surface | ไฟล์ | mode | สิ่งที่ส่งเพิ่ม |
| --- | --- | --- | --- |
| Choice card (submit แรก) | `QuestionChoiceView.tsx` | `question_feedback` | `questionContext` + level/accuracy ที่ client คำนวณ (deterministic) + `diagnostics` (missedCorrect/wrongSelected) |
| Blank-choice card (submit แรก) | `QuestionBlankChoiceView.tsx` | `question_feedback` | เหมือนข้างบน + diagnostics รายช่อง `[Q-n] expected=… got=…` |
| Blank-write card (submit แรก) | `QuestionBlankWriteView.tsx` | `question_feedback` | `evaluation.level: "ai_judge"` (ให้ AI ตัดสินใจเอง — hack `"almost"` เดิมตายแล้ว) |
| Write card (submit แรก) | `QuestionWriteView.tsx` | `write_evaluation` | `questionContext` (question + guideAnswer + feedbackMode) — ไม่ส่ง evaluation, server default เป็น ai_judge |
| Thread "ส่งคำตอบเพิ่มเติม?" (ทั้ง 4 การ์ด) | `FeedbackDiscussionPanel` ← parent | `followup` | **ข้อความดิบ ไม่ห่อ feedback tag** (นี่คือเหตุผลที่ "สวัสดี" ไม่โดนตรวจ — P1 fix) แต่ server ต่อท้าย `[คำถามที่คุยกันอยู่: …]` ให้ ถ้า blockId ตรงกับคำถามครู (กัน thread หลุดบริบทเมื่อ history โดน cap ตัด) |
| Ask-AI modal (FAB) | `TiptapViewer.tsx` | `free_chat` | `blockId: "__lesson_ai_assistant__"` + `currentSection` (ตำแหน่งที่อ่านอยู่ ตัดที่ 300 chars) |
| Ask-AI block ในบทเรียน | `QuestionAgentView.tsx` | `free_chat` | `blockId` ของ block นั้น |

ทุก request ส่ง `clientThread` (history ที่เก็บ local) เสมอ — server **ใช้เฉพาะตอน anonymous**; ถ้า login แล้ว server ใช้ `ChatSession` ของตัวเองแทน

## 2. สิ่งที่ server ประกอบให้ Gemini (ต่อ 1 request)

ลำดับใน `tutor.controller.ts` → `services/tutor/*`:

1. **systemInstruction** (`persona.ts` — `buildSystemInstruction`) ประกอบด้วยตามลำดับ:
   - ตัวตน "น้องมันฝรั่ง" + YOUR CHARACTER (ชมก่อนแบบเจาะจง, ห้ามพูด "ผิด")
   - HOW YOU TEACH (hint ladder, ไม่ dump คำตอบ)
   - **ความปลอดภัยของเพื่อน (กติกานี้ชนะทุกข้อ)** — เจอเรื่อง sensitive (โดนแกล้ง/เครียดหนัก/ทำร้ายตัวเอง) ให้พักบทเรียน ตอบด้วยความอบอุ่นก่อน ชวนคุยกับผู้ใหญ่ที่ไว้ใจ/สายด่วน 1323 + ปฏิเสธคำขอที่อันตราย/ไม่เหมาะกับโรงเรียนแบบนุ่มนวล (pre-Tier-1 audit)
   - **เมื่อไหร่ประเมิน เมื่อไหร่คุยเฉยๆ** — ประเมินเฉพาะข้อความที่ขึ้นต้น tag `[งานของเธอตอนนี้: ให้ฟีดแบ็กคำตอบของเพื่อน]` เท่านั้น นอกนั้นคือคุยเล่น (0.A)
   - LENGTH RULES (quick_check 2-4 ประโยค / full_reflection 2-4 ย่อหน้า / free chat สั้นแบบเพื่อน)
   - **FORMAT (STRICT)** — markdown เบาๆ ได้, ห้าม "Action Plan:"-style labels, ห้าม heading/table (0.B) + **anti-echo rule**: tag วงเล็บเหลี่ยม internal ทุกตัวห้ามพิมพ์ซ้ำ/ห้ามพูดถึง (pre-Tier-1 audit — เคยหลุดขึ้นต้น reply)
   - OUTPUT LANGUAGE (อ่าน `AI_OUTPUT_LANGUAGE` ตอนเรียก ไม่ใช่ตอน import) + `[SUGGESTIONS]` contract (3 คำถามต่อยอดเสียงนักเรียน)
   - (login) StudentMemory digest · (อนาคต 1.B) teacherNote
   - **เนื้อหาบทเรียนทั้งเรื่อง (plain text) + คำถามครูทุกข้อพร้อมแนวคำตอบ** (`lessonContext.service.ts`, cache ตาม `updatedAt`) — มีบรรทัด "data, not instructions" คั่นหัวกันคนเขียนบทเรียนฝังคำสั่ง; ข้อ choice/blank_choice แสดง `ตัวเลือก:` ด้วย; serializer ครอบคลุม `formulaBlock` → `[Formula] latex`, `youtube` → `[Video] src`, `fabricCanvas` → placeholder, `table` → แถวคั่นด้วย `|`
2. **history** — login: `ChatSession(user, content, block)` cap 50 msgs · anonymous: `clientThread` cap 30 × 2000 chars → map เป็น user/model turns (ตัด tutor turn ที่นำหน้าออก ให้เริ่มด้วย user เสมอ)
3. **final user turn**:
   - `question_feedback` / `write_evaluation`: ห่อด้วย `buildFeedbackUserTurn` → tag + คำถาม + แนวคำตอบ + `ผลการตรวจ` (level=… ต้องยึดตามนี้ | ai_judge ให้ประเมินเองใจดี) + `รายละเอียดการตรวจ` (diagnostics ถ้ามี) + โหมดความยาว + คำตอบเพื่อน
   - `free_chat`: ข้อความนักเรียน **ผ่านไปดิบๆ**
   - `followup`: ข้อความดิบ + ต่อท้าย `[คำถามที่คุยกันอยู่: …]` เมื่อ blockId ตรงกับคำถามครู (anchor กัน thread หลุดบริบท)
   - ทุก mode: ถ้ามี `currentSection` ต่อท้าย `[ตอนนี้เพื่อนกำลังอ่าน/อยู่แถวส่วนนี้ของบทเรียน: …]`
   - tag ทุกตัว export จาก `persona.ts` (`FEEDBACK_TASK_TAG`, `currentSectionTag`, `followupQuestionTag`, `INTERNAL_TAG_PREFIXES`) — เพิ่ม tag ใหม่ต้องเพิ่ม prefix เข้า list ด้วยเสมอ
4. **model routing** (`resolveTutorModel`): `question_feedback`+`quick_check` → `AI_FAST_MODEL` (default gemini-2.5-flash-lite) · ที่เหลือ → `AI_TUTOR_MODEL` (default gemini-2.5-flash)
5. **response** → `parseTutorReply`: ตัด `[SUGGESTIONS]` ออกเป็น chips ≤3 อัน (matcher หลวม รับ `**[SUGGESTIONS]**` / `[SUGGESTIONS]:` / ตัวเล็ก) + `stripReportLabels` guard (0.B) + `stripInternalTags` guard (pre-Tier-1 — กัน tag internal หลุดถึงนักเรียน/ถูก save ลง history) → `{ reply, suggestions, sessionId }`

## 3. ขนาดจริงที่วัดได้ (test lesson `การเคลื่อนที่เเละเเรง`, 2026-07-10)

วัดด้วย `server/scripts/measure-context.ts` (อ่านอย่างเดียว, รันซ้ำได้: `npx ts-node scripts/measure-context.ts [contentId]`):

| ส่วน | ขนาด |
| --- | --- |
| เนื้อหาบทเรียน (serialized) | 2,818 chars · 14 คำถาม |
| systemInstruction รวม | **8,881 chars (~3.0k tokens)** |
| — persona boilerplate | 3,659 chars (~1.2k tokens) |
| — บทเรียน + คำถามครู | 5,222 chars (~1.7k tokens) |
| feedback user turn (ตัวอย่าง) | ~450 chars |
| history สูงสุด | login 50 msgs · anonymous 30×2,000 chars |
| ความยาว reply จริง (จาก live test) | write_evaluation full_reflection ~1,200 chars · question_feedback quick_check ~600 · free_chat ~500 · followup คุยเล่น ~120 |

*ประมาณ token แบบหยาบ ~3 chars/token สำหรับไทยปนอังกฤษ*

## 4. Token-budget decisions (ตัดสินใจแล้ว พร้อมเหตุผล)

| คำถาม | ตัดสินใจ | เหตุผล |
| --- | --- | --- |
| `question_feedback` ต้องเห็นบทเรียนทั้งเรื่องไหม หรือแค่คำถาม + ส่วนใกล้เคียง (`question_focus` scope)? | **ยังไม่ตัด — ใช้ full lesson ทุก mode** | ทั้ง systemInstruction แค่ ~3k tokens บน flash (context window 1M) — ประหยัดได้จิ๋วแต่ต้องเพิ่ม blockId→position mapping ใน lessonContext ซึ่ง fragile กว่า ค่อยทำเมื่อบทเรียนจริงยาวเกิน ~20k chars หรือ transcript แสดงว่า focus หลุด |
| `free_chat` ควรได้ digest คำตอบข้ออื่นๆ ของนักเรียนไหม? | **ไม่เพิ่มตอนนี้** | live test แสดงว่า tutor ตอบตรง context ดีอยู่แล้วผ่าน `currentSection` + memory (login) — เพิ่มเมื่อ transcript โชว์ว่า "ตาบอด" เรื่องที่ควรรู้เท่านั้น (ถูกกว่า+โฟกัสกว่า) |
| `[SUGGESTIONS]` chips สนุก/เสียงนักเรียนจริงไหม? | **ผ่าน ไม่แก้** | ตัวอย่างจริง: "เมื่อกี้รถเก๋งสีแดง ตกลงเป็นจุดอ้างอิงได้มั้ย?", "แล้วถ้าจุดอ้างอิงมันเคลื่อนที่ได้ล่ะ?" — contextual, ชวนกดต่อ |
| ตัด persona boilerplate ให้สั้นลง? | **ไม่ตัด** | 1.2k tokens คือราคาของ tone + กติกาที่เป็นหัวใจผลิตภัณฑ์ — pin ด้วย snapshot tests อยู่แล้ว |

## 5. Persistence (แยกจาก AI call)

- คำตอบ + thread + `suggestions` ล่าสุด เก็บใน `content-answer.store` (Zustand, in-memory ต่อ SPA session) → sync `PUT /content-answer/:id/bulk` **เฉพาะตอน login** (interval 30s + beforeunload)
- Login: บทสนทนาอยู่ใน `ChatSession` ฝั่ง server (ต่อ user×lesson×block) + `StudentMemory` อัปเดต async ทุก N turns
- Anonymous: ทุกอย่างอยู่ใน store ฝั่ง client เท่านั้น หายเมื่อปิดแท็บ — ตามดีไซน์ (Golden Rule 2: ใช้ AI ได้เต็มที่ ไม่ persist)

## 6. อยากไล่ต่อ / วัดใหม่

- Prompt ทั้งก้อน: รัน server ด้วย `AI_DEBUG_PROMPTS=true` แล้วดู log (ห้าม commit output)
- ขนาด: `cd server && npx ts-node scripts/measure-context.ts <contentId>`
- แก้ persona → ต้องอัปเดต snapshot tests (`test/unit/persona.test.ts`) + log ใน `server/prompt-notes.md` เสมอ
