# Assignment 3 Report

- **Name**: Samer Roz
- **Student ID**: TODO
- **Group**: TODO (BBADBA 5A or 5B)

## Use case

I run talks., an anonymous social app for university students that's live at IE University. The people asking this assistant are me, our devs and our student moderators. The questions are things like "a user wants all their data deleted, do we have to?" or "what do we have to tell someone when we remove their post?".

They act on the answer straight away. They delete data, take posts down, or change a feature. A wrong answer can mean a GDPR fine, a post removed without the statement of reasons the DSA requires, or ignoring a duty we actually have. "The corpus doesn't cover this" just costs us a message to a lawyer. So I'd rather it refuse than guess, and I built everything below around that.

## Corpus

| Document | URL | Date | Licence | Words (articles) |
|---|---|---|---|---|
| GDPR, Regulation (EU) 2016/679 | https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016R0679 | 2016-04-27 | EU legal acts, free reuse with attribution (Commission Decision 2011/833/EU) | 29,094 |
| Digital Services Act, Regulation (EU) 2022/2065 | https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32022R2065 | 2022-10-19 | same | 32,151 |
| ePrivacy Directive 2002/58/EC, consolidated 19 Dec 2009 | https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:02002L0058-20091219 | 2009-12-19 | same | 4,714 |
| Total | | | | 65,959 |

I indexed the articles only and left out the recitals and annexes on purpose. Two reasons. The free Gemini tier gives about 1,000 embeddings a day, and with recitals the three acts come to roughly 124k words, which would have blown the budget once I embedded two chunkings. And recitals explain the law but don't create duties, and my users want to know what they have to do.

That choice has a cost, and my test set shows it. q39 asks whether a feed only open to verified IE students counts as "the public" under the DSA. The answer is basically in recital 14, which isn't indexed, so that question is unanswerable for my system.

## Ingestion

The texts come from the EU Publications Office (Cellar) as XHTML. EUR-Lex itself blocks scripted downloads with a challenge page, but Cellar serves the same official text, so `main.py index` downloads everything with no manual steps.

`index.py` finds each article by its id (`art_17`), takes its title and chapter, and splits the body into numbered paragraphs.

The extraction problem I fixed: the Official Journal lays out lists as two-column tables, so "(a)" and its text came out on separate lines and the label floated away from its sentence. I join a label to the line after it. The consolidated ePrivacy text also has amendment markers like ▼M2 in the middle of sentences, which I strip, along with footnote markers.

The GDPR's definitions (Article 4) are one giant paragraph, so any paragraph over 300 words gets cut into 300-word windows with 30 words of overlap.

Each chunk stores its source, its location (like "Article 17(1)-(2)"), the article numbers it covers, a heading, a text hash and a word count. The act name and article title go at the top of the chunk text so the embedding knows what it's reading.

| Design | GDPR | DSA | ePrivacy | Total |
|---|---|---|---|---|
| fixed (300 words, 50 overlap) | 119 | 131 | 20 | 270 |
| article (one per article, split on paragraphs past 300 words) | 159 | 162 | 30 | 351 |

Vectors are saved in `index/vectors.jsonl`, keyed by a hash of the text and appended after every batch of 50. If the run dies or hits the daily quota, nothing already embedded is lost and nothing gets embedded twice. Query vectors are cached the same way in `index/query_vectors.jsonl`.

## Test set

The test set was written before I tuned anything.

I wrote a prompt with the talks. context (anonymous feed, moderators, the First Connection questionnaire, ads coming, some 17 year old freshers, micro company) and the rules for good questions. An AI agent drafted a longlist of 45 from that. A second agent re-checked every row against the law text without seeing the first one's reasoning. I dropped the rows that had an error or leaned on a judgment call instead of the text, and I dropped everything about our dating feature to keep it on the core app.

The final set has 40 questions: 25 single, 7 multi, 8 unanswerable.

Two examples:

- q17 (single, dsa, Article 19): "We have under 10 people and under €2 million turnover. Does Article 19 of the DSA let us skip all the online platform rules?" The reference answer is that Article 19 switches off Articles 20 to 28 except 24(3), but not the duties every hosting service has, like notice and action and statements of reasons.
- q33 (unanswerable): "How fast do we have to take down terrorist content after an authority orders it?" The answer lives in a regulation I deliberately didn't index (2021/784), which I use for the add-document demo.

## Retrieval

A hit means a top-k chunk is from the expected act and covers the expected article. For multi questions both articles have to be in the top k. These numbers are on the 32 answerable questions.

| Design | hit@1 | hit@3 | hit@5 | single hit@5 | multi hit@5 | words added (top 5) |
|---|---|---|---|---|---|---|
| fixed | 0.41 | 0.62 | 0.75 | 0.84 | 0.43 | 1,527 |
| article | 0.56 | 0.78 | 0.91 | 0.96 | 0.71 | 1,019 |
| article_hybrid (dense + BM25, RRF) | 0.31 | 0.47 | 0.59 | 0.68 | 0.29 | 1,153 |
| article_routed ("Article N" in the query pulls that article up) | 0.56 | 0.78 | 0.91 | 0.96 | 0.71 | 1,019 |

Article chunks beat fixed windows at every k, and they put about a third fewer words in the prompt. Fixed windows cut an obligation in half or glue the end of one article to the start of the next, so the chunk that matches the question often doesn't hold the whole rule.

Hybrid was the worst, which I didn't expect, so I ran BM25 on its own to see why. Alone it gets 0.25 at hit@5. Its favourite results are short chunks that happen to share an everyday word with the question: ePrivacy Article 9 shows up in the top 5 for 12 of the 32 questions, and "A student emailed asking for a copy of everything we hold on them" pulls ePrivacy's email marketing rules and DSA Article 25 instead of GDPR Article 12. My users ask in everyday words and the law uses its own, and the embedding bridges that gap where keyword matching can't. Fusing a ranking that weak into the dense one dragged the right article down.

Routing tied with plain article chunks because only two questions name an article, and plain search already finds both. So I serve plain article chunks. The extra rule isn't earning anything on my test set.

With 32 questions, one question moves hit@5 by 0.03, so I'd only read the big gaps (article vs fixed vs hybrid) as real.

Multi questions are the weak spot for a single search (0.71). The agent can search more than once, though, and across its searches it retrieved the expected article for 31 of 32 answerable questions.

## Tools and subagent

`server.py` exposes two tools.

`search(query)`: "Search EU law that applies to a student-run anonymous social app: the GDPR, the Digital Services Act and the ePrivacy Directive..." It returns the top 5 passages, each with its chunk id, act, article and similarity, and it tells the model that similarity under about 0.55 usually means the corpus doesn't cover the question.

`add_document(path)`: "Add one document to the search index from a local file path... If the identical file is already indexed it is skipped and nothing is embedded. If a document with the same name but different content is indexed, its old chunks are replaced."

One real search result, for "Do we have to tell a user why we removed their post?":

```
[dsa-article-0026] DSA Article 17(1)-(2) (similarity 0.72)
1. Providers of hosting services shall provide a clear and specific statement of reasons to any
affected recipients of the service for any of the following restrictions imposed on the ground that
the information provided by the recipient of the service is illegal content or incompatible with
their terms and conditions: (a) any restrictions...
```

The subagent (`.claude/agents/corpus-researcher.md`) can only use `mcp__corpus__search`. Its instructions: always search first, search up to twice more with the legal term if results look off, answer only from the passages, cite every claim by chunk id, say exactly "The corpus does not cover this." when it isn't there, keep it to 2 to 6 sentences, and add no national law or numbers that aren't in the passages.

Transcript from Claude Code:

```
TRANSCRIPT TO PASTE
(run in the repo: claude, then "Use the corpus-researcher subagent: a user reported a post as
defamatory, what do we have to do?" and paste the full exchange here)
```

## Adding a document

`add_document` hashes the file (sha1) first. If a document with the same name and the same hash is already in `index/documents.json`, it returns "already indexed with identical content. Nothing added." and embeds nothing. If the name matches but the hash doesn't, say an amended version of a law, it deletes that document's old chunks and adds the new ones, so a stale version never sits next to the current one.

The demo (`demo_add_document.py`, output in `eval/add_document_demo.md`) uses q33, the terrorist content question.

Before: the agent found the DSA's Article 9, which only says you inform the authority "without undue delay" of the effect given to an order, and said the corpus has no specific deadline for terrorist content.

Then I asked the agent to add the Terrorist Content Online Regulation (2021/784). With approval denied, it said the document wasn't added. With approval given, it added 35 chunks. Calling `add_document` on the same file again returned "already indexed with identical content".

After: "You must remove terrorist content or disable access to it as soon as possible, and in any event within one hour of receiving a removal order from a competent authority [tco-article-0004]", citing TCO Article 3.

The script then removes the TCO chunks again (the vectors stay cached, so re-adding costs nothing). I do this so `main.py evaluate` keeps measuring the corpus the test set was written for. Otherwise q33 would stop being unanswerable and I wouldn't notice.

## Harness and guardrails

`agent.py` uses LangChain's `create_agent` on gemini-3.5-flash-lite with the MCP tools and `AGENTS.md` as the system prompt. The rules in short:

- answer only from passages search returned this turn
- always search first, and retry with the legal term
- cite every claim by chunk id
- say "The corpus does not cover this." when it doesn't
- give every condition and exception, and check for a micro or small enterprise exemption
- answer the covered part of a partly covered question and say which part isn't
- no national law, case-specific fines or numbers that aren't in the passages
- 2 to 6 sentences
- a lawyer line only when there's real legal risk
- add documents only when asked, then just report what happened

Each rule says why. The top of the file states the facts about talks. that change answers: micro enterprise, not a very large platform, some 17 year old users.

The approval step is a `wrap_tool_call` middleware. Before `add_document` runs, it asks on the terminal (or asks an approver function in code), and if there's no terminal it refuses. In the demo the denied request didn't add anything and the approved one did. There are also call limits: 6 tool calls and 8 model calls per question.

My extra guardrail is a citation check after the agent answers. Every chunk id in the answer must be one that search returned in this run, and an answer with no citation at all must be a refusal. It doesn't check that each sentence has a citation, so an answer with one real citation and one uncited claim still passes (q17 does this). If either fails, the model gets one retry with the problem spelled out. If it fails again, the answer is withheld and replaced with a refusal.

Where it fired: in run 2 on q04, the first answer cited `gdpr-article-0042`, which search never returned. On the retry it cited `gdpr-article-0012`, GDPR Article 7, the real passage. In run 1 it fired on q26 and q27 and withheld both, so two answers that may have been right became refusals.

Run 1 didn't save the withheld text, so I can't confirm why those two failed. My best guess was that the model put several ids in one bracket and my regex missed them. It could also have been the start-only refusal check, which I changed at the same time. I changed the regex to find ids anywhere and started logging the first answer. In run 2 both questions came back with valid citations.

So the guardrail turns made-up citations into refusals, which is the cheap mistake for my users. When it misfires, it turns right answers into refusals too.

## Evaluation

What code checks (`evaluate.py`):

- did it refuse, and should it have
- are all cited ids among the ones search returned
- did the agent retrieve the expected article
- did it cite it
- did the guardrail fire
- tokens, tool calls and seconds

The judge grades two things with my rubric. Correct: it states the substance of the reference answer with no contradiction; missing a main condition or half of a multi question is 0; for unanswerable, it says the corpus doesn't cover it and doesn't answer anyway. Supported: every claim about the law is in the evidence passages, and a number, deadline or national rule that isn't there makes it 0. The judge sees the question, the reference answer, the answer and the retrieved passages.

Judge model: I planned gemini-3.5-flash, but its free tier allows 20 requests a day and testing used them up. So the judge is gemini-3.5-flash-lite, the same model as the agent. A model grading its own family can go easy on it, and I found one case by hand: in q27 the judge marked as correct an answer that puts the exemption in "Section 2" of the DSA, when Article 19 sits in Section 3.

Agreement with me: PENDING. My 20 blind grades (odd ids, q01 to q39) go in `eval/human_labels.csv`, and `evaluate.py` recomputes agreement from that file.

As a stand-in until then, I had a different model (Claude) grade the same 20 answers blind to the judge, with the same rubric, in `eval/ai_second_grader.csv`. This is not a human check. It agreed with the judge on correct for 18 of 20, but on supported only 14 of 20. It marked 7 of the 20 as unsupported, where the judge marked 2 of all 40. Reading its notes, the pattern is real: answers overstate the source. They turn an "as appropriate" security measure into a hard duty (q11), present the condition for keeping the liability exemption as a standalone duty to remove (q13), and talk about "Section 3" when the chunk only says "this Section" (q29). So I don't trust the judge's 2/40 unsupported number. It's lenient on correctness too: the second grader failed q17 and q27, which the judge passed, so a stricter count is 27/32, not 29/32.

| | Run 1 | Run 2 |
|---|---|---|
| Correct, answerable (judge) | 25/32 | 29/32 |
| Unsupported answers (judge) | 0/40 | 2/40 |
| Correct refusals (code) | 6/8 | 7/8 |
| Wrong refusals (code) | 2/32 | 1/32 |
| Citations all valid | 40/40 | 40/40 |
| Guardrail fired | 2 | 1 |

Between the runs I changed five things, all picked after reading run 1's failures on these same 40 questions:

1. the citation regex
2. the micro enterprise facts and the "every condition and exception" rule in `AGENTS.md`
3. a shorter lawyer line
4. the code's refusal check, which now counts "the corpus does not cover" (or "state", "specify" and similar) anywhere in the answer instead of only at the start
5. the served design, from article_routed to article (identical retrieval scores, so probably harmless)

So run 2 is optimistic. I tuned on the test, and only fresh questions would show whether the gain is real. I also never reran run 2 unchanged, so I don't know how much a rerun moves the score by itself. And q24 was right in run 1 and became a wrong refusal in run 2, so at least one fix broke something. Tuning didn't only add. The refusal-check change also means the two runs' refusal counts aren't measured in exactly the same way.

## Failures

My hand classification of run 2, from reading each answer against the reference and the passages:

| Cause | Count | Questions |
|---|---|---|
| Unsupported claim | 2 | q39 (answered an unanswerable question with its own reading of "the public"), q24 (a refusal that also claims ePrivacy is missing from the index, which is false) |
| Wrong answer from the right passage | 3 | q04 (had GDPR Article 7 but only said withdrawal must be as easy as giving consent, and left out "at any time" and being told beforehand), q27 (skipped Article 20 entirely, and "Section 2" is wrong), q17 (the judge passed it, but it calls Section 3 "transparency obligations" and never says the hosting duties in Articles 16 and 17 still apply) |
| Wrong refusal | 1 | q24 (DSA Article 10 was retrieved, and it still refused) |
| Retrieval miss | 1 | q28 (searched once, never retrieved Article 19, so it told a micro company it "must" follow Article 28, which Article 19 exempts it from) |
| Bad test question | 0 | |

q24 is counted twice because it failed in two ways. On top of these, the second grader's notes point to smaller overstatements in answers the judge passed (q11, q13, q29, q33). Those don't change the conclusion of the answer, but a moderator reading "you must" where the law says "as appropriate" would act on a stronger duty than the one that exists.

The code's first guess called q37 an "unsupported claim", and I think that's wrong. The answer says the corpus doesn't specify a fixed number of hours and then gives DSA Article 6's "act expeditiously" with a citation, which is my reference answer, and both the judge and the second grader marked it correct. My refusal check looked for "does not cover", "does not state" and "does not say", so it missed "does not specify". That's a measurement bug, not an agent failure. I fixed the check and recomputed from the saved answers (no new model calls), and correct refusals went to 7/8.

What I'd fix first is exemptions. Four of the seven multi questions (q27, q28, q29, q32) hinge on Article 19, plus q17, and q28 still missed it in run 2 because the agent searched once. I'd force a second search whenever a retrieved article sits in DSA Articles 20 to 28, and put the section number in the chunk header so "Section 3" is in the text and the model can't make up "Section 2". Second, I'd add a rule to keep the law's own qualifiers ("as appropriate", "where", "unless") word for word.

## Decision

I'd let our moderators and devs rely on it for a first pass: which article applies to this situation and what does it say, with the citations right there to click. It's mostly right on single-article questions, though it sometimes states a duty more strongly than the law does, and it's good at saying when something isn't covered.

I wouldn't let anyone act on it alone when the answer hinges on whether an exemption applies to us (Article 19 and friends). That's where it fails most, and a wrong call there costs the most. q28 is the example: it told us we "must" follow Article 28 when we're exempt. I also wouldn't use it for a regulator letter, a police order or a court order without a lawyer, and the agent already says so in those cases. And because the judge is lenient on support, whoever reads an answer should click the citation before acting on a "you must".

The safeguards I'd keep:

- citations always shown
- the citation guardrail
- refusal when the corpus doesn't cover something
- a weekly look at a sample of answers by me

Before trusting the run 2 number I'd want 20 or so fresh questions written after the fixes, and my own grades in.

Cost per question in run 2: 5,354 input and 143 output tokens and 1.4 tool calls on average. That's up from 4,235 tokens and 1.2 calls in run 1, probably because of the longer AGENTS.md and more searches. Mean 7.2 seconds, worst 45.5 seconds (that one waited on rate limits). On the free tier that's €0 per question. I haven't priced the paid tier here, so I won't put a number on it.

## AI use

I used Claude (Anthropic) throughout. It wrote the code (`index.py`, `retrieval.py`, `server.py`, `agent.py`, `evaluate.py`, the demo script), ran the indexing and the evaluations, and found the EU Publications Office route when EUR-Lex blocked downloads.

For the test set, I gave an AI agent a prompt with talks.' features and my criteria, it drafted 45 candidates, and a second agent checked each against the law. I set the criteria, cut the dating questions and approved the final 40.

Claude also ran an independent AI grading pass, labelled as such above, and drafted this report from the eval files.

The decisions are mine: the use case and corpus, indexing articles only, serving plain article chunks, refusing over guessing, the rubric, the fixes after run 1 and the ship decision.

My 20 human grades are listed as pending above until they're in.
