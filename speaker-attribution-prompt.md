# Speaker Attribution Prompt

You are labelling speakers in a transcript of an outbound sales call from Kotak Mahindra Bank about a pre-approved personal loan.

Two speakers:
- **AGENT** — the bank telecaller who placed the call
- **CUSTOMER** — the person who answered

The transcript is ASR output in romanized Hinglish. It contains recognition errors, garbled words, and repeated words. Do not fix them — label each line as it is.

## Step 1 — Anchor on the opening

The call always begins with the agent's script:

> "Good afternoon, this is `<NAME>` calling from Kotak Mahindra Bank. Am I speaking with `<NAME>`? / Kya meri baat `<NAME>` ji se ho rahi hai? This call is recorded for internal training and quality purpose."

The bank name is often garbled — *Kotak 500, Medra, Ventra, Maintha, Mainina, paandr, mahendr, kodak*. Treat any `kotak`/`kodak` variant the same.

These opening lines are known to be the AGENT. Use them to fix two patterns for the rest of the call:

- **Honorific** — whichever of `sir` or `ma'am`/`madam` the agent uses. That one marks agent lines; the other marks customer lines.
- **First-person verb form** — either the `-oonga / raha hoon / karta hoon / sakta hoon` family or the `-oongi / rahi hoon / karti hoon / sakti hoon` family. Lines using the agent's form are AGENT; lines using the other are CUSTOMER.

If both speakers use the same honorific or the same verb form, that signal carries no information for this call. Skip it, lean on rules 4–11, and mark more lines low confidence.

## Step 2 — Label every line

Signals in order of strength. Earlier ones override later ones.

**1. Script lines are AGENT.** Greeting, self-identification, recording disclosure, and closing (*"thank you for your valuable time"*, *"have a nice day"*, *"kimti samay dene ke lie dhanyavaad"*).

**2. Honorific match.** Apply the mapping fixed in Step 1.

**3. Verb-form match.** Apply the mapping fixed in Step 1. Independent of the honorific, and often survives when the honorific is garbled.

**4. Possessive direction — whose loan is it.** When the speaker treats the loan, offer, account or EMI as belonging to the *listener* (`aapka`, `aapko`, `aapke paas`, `your loan amount`), the speaker is the AGENT. When the speaker treats it as their *own* (`mera`, `mere paas`, `mujhe`, `main`), the speaker is the CUSTOMER. Examples: *"aapko ek lakh ka offer hai"* → AGENT. *"mere paas already loan chal raha hai"* → CUSTOMER.

**5. Polite imperatives are AGENT.** `-kijie`, `-dijie`, `-lijie`, `-ie`: *click kijie, open kijie, daal dijie, bataie, dekhie, rukie*. Informal imperatives aimed at the other party (*kar do, batao, bata do*) are CUSTOMER.

**6. Instruction vs. screen report.** During app walkthroughs the agent instructs and the customer reports what appears. *"Us par click kijie"*, *"loan and card offer mein jaie"* → AGENT. *"Kar diya"*, *"khol diya"*, *"aa gaya"*, *"dikhai nahin de raha hai"*, *"statement nahin"* → CUSTOMER.

Related: the agent reads the offer off their own system early in the call — *"jaise ki main dekh pa raha hoon aapko ek laakh ka offer hai"* → AGENT.

**7. Question type.**
- Asking about the product → CUSTOMER: *"Rate of interest kya hai?"*, *"ROI kitna hai?"*, *"processing fee kitna hai?"*, *"pre closure kya hai?"*, *"chhah mahine ka kitna hoga?"*
- Verifying, driving the process, or checking the line → AGENT: *"Am I speaking with X?"*, *"can I explain your loan offer sir?"*, *"aapko kya dikhai de raha hai?"*, *"Am I audible sir?"*, *"aapka do minute lagega?"*

**8. Right-party confirmation and OTP.** The agent asks for the last 3 digits of the mobile number and instructs on the OTP (*"last ka 3 digit bataie"*, *"OTP daal dijie"*, *"OTP kisi ko mat bataiye"*). A line that is just a bare digit string (*"083"*, *"873"*) is the CUSTOMER answering. Reporting receipt (*"OTP aaya"*, *"OTP check kar raha hoon"*) is the CUSTOMER.

**9. Product explanation is AGENT.** Loan amount, EMI figures, tenure options, interest rate, processing fee, GST, stamp duty, insurance, top-up eligibility, foreclosure charges, documentation requirements. Usually the longest lines in the call.

**10. Objections, refusals and deferrals are CUSTOMER.** *"interest bahut zyaada hai"*, *"main meeting mein hoon"*, *"I am not interested"*, *"main kisi aur bank se le loonga"*, *"shaam tak call karna"*, *"main branch jaakar karunga"*. Naming another bank as their own account or loan (*"HDFC mein hai ji"*, *"SBI aur ICICI"*, *"doosre bank mein kam mil raha hai"*) is also the CUSTOMER — though the agent *asking* whether they hold another account is the AGENT.

**11. Weak priors, only when nothing above applies.** Long explanatory lines lean AGENT. Very short acknowledgements (*haan, ji, ok, achchha, thik hai*) lean CUSTOMER. Conversation mostly alternates.

## Step 3 — Common traps

- **Both speakers use the same acknowledgements** — `haan`, `ji`, `ok`, `achchha`, `thik hai`, `bilkul`. These carry almost no signal on their own; decide from the surrounding turns.
- **The agent repeats the customer's words back** when confirming. A phrase appearing twice does not mean the same speaker said it twice.
- **`main` alone does not mean CUSTOMER.** The agent says *"main aapko bataana chaahoonga"*. Rule 4 is about whose *loan* is being discussed, not who says "I".
- **A line may contain both speakers** where the ASR dropped the boundary. Label it with the dominant speaker and mark confidence low.

## Step 4 — Special cases

- **`SYSTEM`** — automated audio: voicemail prompts (*"if you record your name and reason for call"*, *"we are unable to connect with you"*, *"please stay on the line"*), IVR, hold, and any `[PAUSE]` marker. If a second greeting appears mid-transcript, another dial attempt was concatenated — re-run Step 1 from that point.
- **`UNKNOWN`** — the line is too garbled to attribute, or signals genuinely conflict. Prefer this over guessing, but keep it under roughly one line in ten.

## Output

Return JSON only, no commentary. Every input line number appears exactly once.

```json
{
  "agent_honorific": "sir | ma'am | null",
  "agent_verb_form": "oonga | oongi | null",
  "labels": [
    {
      "line": 1,
      "speaker": "AGENT | CUSTOMER | SYSTEM | UNKNOWN",
      "confidence": "high | medium | low",
      "signal": "short phrase, e.g. 'opening script', 'honorific match', 'verb form', 'possessive direction', 'screen report'"
    }
  ]
}
```

## Worked example

```
1. Good afternoon, this is Meera calling from Kotak Mahindra Bank.
2. Kya meri baat Anil ji se ho rahi hai?
3. Haan ji.
4. Sir aapko ek lakh ka pre approved personal loan offer hai, main bataana chaahoongi.
5. Rate of interest kitna hai madam?
6. Bahut zyaada hai, HDFC mein kam mil raha hai mera.
7. Sir aap app open kijie.
8. Khol diya.
```

Anchors from lines 1–2: agent honorific `sir`, agent verb form `oongi`.

1 AGENT (opening script) · 2 AGENT (opening script) · 3 CUSTOMER (short acknowledgement) · 4 AGENT (honorific + verb form + possessive direction) · 5 CUSTOMER (product question + other honorific) · 6 CUSTOMER (objection + own bank) · 7 AGENT (honorific + polite imperative) · 8 CUSTOMER (screen report)

---

## Transcript

```
<<< PASTE NUMBERED LINES HERE >>>
```
