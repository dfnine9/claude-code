# Daniel Fatemi — Resume 2026

Source-controlled resume. `Daniel_Fatemi_2026.html` is the single source of
truth; `Daniel_Fatemi_2026.pdf` is the rendered output.

## What changed

Added **cila.ai** as the lead entry under *Selected AI Projects*, written to
match the existing voice and bullet style (strong opening verb, dense
technical detail, hard metrics, em-dash clauses, "owned end-to-end").
The skills section gained the technologies that project demonstrates
(Next.js, PostgreSQL/Supabase, Stripe, pgvector, AES-256-GCM/HMAC envelope
encryption).

The layout is a faithful reconstruction of the original PDF: Liberation
Serif (Times-compatible), navy `#0a2540` small-caps section headers with
rules, justified body, navy `▸` arrows for projects/education and `•` for
experience, and the two-column skills table.

## Regenerate the PDF

```bash
# one-time deps (Debian/Ubuntu)
apt-get install -y poppler-utils
pip install weasyprint

# render
python3 -m weasyprint resume/Daniel_Fatemi_2026.html resume/Daniel_Fatemi_2026.pdf
```
