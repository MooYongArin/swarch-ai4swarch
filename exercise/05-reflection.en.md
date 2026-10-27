**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:**  Arin Ariyata / 6631360821
**My role on the review board:** AI analyst

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I trust AI in the analysis of the documents it's given: find gaps, contradictions and failure cases. For example, the concern where checkout and the growing email backlog share the same durable storage, so a long MailFast outage could fill the storage and stop checkout itself which is the problem AI found and humans had missed. I would keep anything that needs context outside the documents: law, company policy, deadlines, business promises, and past incidents for human. For example, the concern where MailFast stores data in the US; PDPA needs a legal basis for cross-border transfer, legal hasn't approved it, and the ADR has no fallback which a human found, but the AI couldn't have known because that information wasn't in any file it was given.
