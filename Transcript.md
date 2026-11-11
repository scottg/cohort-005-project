# Grilling session: Refund language and the Seat model

Date: 2026-11-11. A `/grill-with-docs` session (grilling + domain modelling) between Scott and Claude. Claude's questions are given in full with its recommendation (➡️); Scott's answers are verbatim.

Outputs of the session: `GLOSSARY.md`, `FUTURE.md`, `IDEAS.md`, `docs/adr/0001-refunds-need-no-approval.md`, `docs/adr/0002-refund-is-the-cancellation-record.md`.

---

## Round 0: Topic

**Q1 - What are we grilling?** Which plan, feature or idea do you want to stress-test?
➡️ Suggested: the full lifecycle of a Refund Request.

**Scott:**
> The language around the refund operation is using multiple terms for the same concept - Chargeback, Reversal, and maybe others. We need to hone in on one term that is commonly used and understandable to users, and ensure we're consistently using that term in the code, in the UI text, and so on.

**Findings before Round 1:** commit `08537a9` had already removed Chargeback and Reversal from `app/`, `scripts/` and migrations, and `GLOSSARY.md` already named Refund as canonical. Remaining drift: the `refunds` table and `RefundStatus` enum actually hold Refund Requests; `resolved_*` columns vs the glossary's "decide"; untermed concepts (refund window, watch/completion limits, refund offer); no code for Pending/Issued/Failed Refunds.

---

## Round 1

**Q1 - Is "Refund" the one canonical term?**
➡️ Yes: Refund for the money, Refund Request for the ask; Chargeback, Reversal, Cancellation forbidden everywhere.

**Scott:**
> Yes, Refund is the word, and should be the root of each var, function name etc. in the code, e.g. RefundRequest, RefundRequested, RefundInProgress, RefundCompleted, RefundApproved, RefundAmount, etc., but "Cancel this Course" is still the set of words we'd use on the UI and maybe in some of the code, e.g. CancelCourse function let's say. A 'cancellation' doesn't always get a refund, if certain conditions are or aren't met. Or we could go with that text on the UI, but the function it triggers being RefundRequested. We will have to track each RefundRequest independently and ensure we aren't having multiple cancellations for the same course, and we need to handle where we have a team purchase with multiple seats where the purchaser can reduce the number of seats and get a partial refund.

**Q2 - Rename the `refunds` table to match the glossary?** The table holds Refund Requests, so `refunds.status === Open` reads as "an open Refund". Options: (a) rename to `refund_requests` / `RefundRequestStatus`; (b) keep; (c) defer.
➡️ (a), rename now.

**Scott:**
> Hmm, Cancellations might be the correct table name, associated with the user who purchased. Refund would then be linked to a Cancellation, which is linked to a 'course' or to a 'course set', or maybe we manage course purchases as 1 or many. refunds.status = open is not clear, refunds.status = requested, approved, rejected, inprocess, completed would be the kind of statuses I'd like to see. But not sure what status values would make sense for partly-approved where they get a partial refund. And again, we need to determine how to manage refunds of an individual's purchase, and refunds for a set of seats, where the refund is for all of them, or partially for reduction in seats wanted.

**Q3 - Should Chargeback stay forbidden, or get its own meaning?** In payments, a chargeback is a bank-initiated pull-back, a different event from a Refund. (a) keep on _Avoid_; (b) define as a distinct concept.
➡️ (b).

**Scott:**
> Chargebak should stay forbidden, it's implied that a chargeback happened when refund.status = complete.

**Q4 - Use "decided" instead of "resolved"?** Rename `resolved_at` / `resolved_by_user_id` to `decided_*`.
➡️ Yes.

**Scott (this answer addressed the eligibility rules, i.e. Q5):**
> I think we need something like a state table that describes refunds based on time, other factors, that we can update as needed. Hard coding it doesn't seem best. E.g. we may have multiple points where the refund is reduced, we could say if you've watch 1/3 and you want a refund, then we refund you 2/3rd of the total cost, you would then only have access up to the videos you watched and paid for, and still have access to the discord for the course. I think we'd want to be nice to users, we could collect each reason and check those regularly to see if we can adjust something later.

**Q5 - Add "Refund Window" and the eligibility rules to the glossary?**
➡️ Yes: Refund Window and Refund Eligibility; leave "Refund Offer" out.

*(Answered by Scott's Q4 answer above.)*

---

## Round 2

Facts found: a team purchase is one `purchases` row with N seat `coupons` pointing at it; a claimed team seat can't currently be refunded; refunds are currently always full.

Note from Claude on Q3: in payments a chargeback is the bank pulling money back, not a completed Refund, which is exactly why the word should stay out.

**Q1 - Can a Refund be partial?** The glossary says "always the full amount".
➡️ Yes, drop "always the full amount".

**Scott:**
> Let's keep it simple and it's either a full refund or none. We can go more granular in the future.

**Q2 - Does a Cancellation ever happen with no Refund?** Scenario: bought 60 days ago, outside every rule, clicks "Cancel this course". (a) loses access, gets $0 (Cancellation is its own concept); (b) told nothing is refundable, keeps access ("Cancel" is UI wording for a Refund Request).
➡️ (b).

**Scott:**
> Check if a user or team is eligible for a refund if they cancel any time we show a page where we would put the cancellation button, and if they are not eligible for any refund, then we don't give them a cancellation button, so there is no path to create a Refund Request We show in the UI that they aren't eligible and why.

**Q3 - Two separate statuses: the decision and the money?** Refund Request: `requested → approved | declined`; Refund: `in_progress → completed | failed`; partial approval shown by amount.
➡️ Yes.

**Scott:**
> Ok, you figure out how to separate those cleanly, and so there cannot be a status that a refund can get into that conflicts somewhere else or would not be possible. State transitions for the refund process need to not get stuck or go into states that are not allowed. But really, if the user is eligible for a refund at that point, why does it need anyone's approval? We want to take us out of the picture for approvals, and have the system determine if they're eligible. If they aren't eligible, then they don't get to cancel anyway, and if they still want to, they can contact directly and ask for a refund for mitigating circumstances, that's when our team would be involved. Yes, use two statuses, and show partial refunds via the amount, not the status.

**Q4 - "Declined" or "Rejected"? "Issued" or "Completed"?**
➡️ Declined and Completed.

**Scott:**
> Use Ineligible for Refund, not Declined or Rejected.

**Q5 - Which team seats can the purchaser give back?** 10 seats, 6 claimed. (a) only the 4 unclaimed; (b) claimed too.
➡️ (a).

**Scott:**
> If partial number of seats is being cancelled, check the number of unclaimed seats and if there are enough, then just do the cancel + refund for those seats. If there are not enough unclaimed seats to perform the requested number of seats cancellation request, then send a note automatically to the team purchaser, let them go in and remove people from seats, and push continue to do the refund, or change it. Kind of like an airline seat map. BTW, ONLY the original purchaser of team seats can request cancellation of seats, if one of their team has claimed a seat, that person should be able to cancel, but that cancellation request will be routed to the team seats purchaser for adjudication and approval, and whether the seat gets cancelled, left unclaimed by removing the user is up to them. We would want a status that indicates cancellation was requested of a team seat, and that we're waiting on the team seats purchaser to respond.

**Q6 - Only one undecided Refund Request per Purchase at a time?**
➡️ Yes.

**Scott:**
> Agree, but scoped to the course in question; if there are other courses they purchased, they should be able to cancel seats in those at the same time as cancelling seats in another course.

**Q7 - Refund rules stored as data, called the "Refund Policy"?** Which version applies, and does the policy only suggest or decide?
➡️ Yes; the policy in force at purchase applies; policy computes, team approves.

**Scott:**
> Yes, the earlier purchase terms prevail, but I think since we're doing online learning, they can cancel at any time before they've watched any of the course content, even if it's after the course start date. We would want to cap that to 30 days or something, so make that a variable we can change later. Add to a FUTURE.md file that we may have terms different for each course.

**Q8 - What access is left after a partial Refund?**
➡️ Keep started lessons and the community ("Partial Access").

**Scott:**
> Good point. Let's do this -- if they only watch the intro section videos, and have not watched any of the chapters after that, then they can still get a full refund. If they've watched more than 4 videos in any section after the intro section/chapter, then that switches of the cancellation ability and by side-effect, no refund happens as it can't be triggered.

**Q9 - "Decided" instead of "resolved"?** (carried over)
➡️ Yes.

**Scott:**
> This might be moot, by my earlier answers. Let's leave this open and if you still think it applies after this round, then come back at me with it.

*Written after Round 2:* `GLOSSARY.md` updated (full-or-nothing Refund, Refund Policy, Refund Eligibility, Ineligible for Refund, In-Progress/Completed Refund); `FUTURE.md` created (per-course policy, partial refunds).

---

## Round 3

Facts found: no "intro section" in code (modules only have `position`); seats are stored as `coupons`; current rules use seconds watched and completion %. Q9 from Round 2 declared moot.

**Q1 - Is "Cancel" UI wording only, with Refund Request as the domain term?**
➡️ Yes.

**Scott:**
> Yes, Cancel is only for the UI. It's binary, if they can cancel, they're getting a Refund of some type. If they can't cancel because they're now ineligible to, then there is no path to create a Refund object in the database for a given user/team seats for a given course. So if a Refund object exists, then a Cancellation occured. We don't need to duplicate that knowledge by creating and managing cancel objects. The Refund object will record the reason for the refund.

**Q2 - Is this one rule the whole Refund Policy?** Eligible if within a 30-day window and no more than 4 lessons watched in any single module after the first.
➡️ Yes.

**Scott:**
> These rules overlap. If they are 30 days post start of a cohort course, then no refund, they have access forever. If they are within the 30 days but have watched 4 complete lessons, then no refund (we do this so they can get familiar and decide part way in if it's for them). For courses that are always available, the same rules apply, but the 30 day clocks starts from when they purchased.

**Q3 - Is the intro section the first module?** (a) position 1; (b) a flag.
➡️ (a).

**Scott:**
> Yes, the first module is always the set of intro videos on how to set up your environment, how we'll proceed, etc. Those should always be free, and open to all, even non purchasers, so they can eval before buying.

**Q4 - What counts as "watched" a lesson?** (a) any playback; (b) at least half; (c) marked complete.
➡️ (a).

**Scott:**
> A watched lesson is having completed watching 4 videos past the intro module. The risk is that a snarky student watches each video and stops each before it ends, so we need to maintain a count of how many videos they've started watching, and how many they completed watching. Let's tie the refund eligibility to if they started and not finished up to 8 videos post-intro module.

**Q5 - Use the names Team Purchaser, Seat, Seat Holder and Unclaimed Seat, and rename `coupons`?**
➡️ Yes.

**Scott:**
> Use terms "Purchaser", "Seat", "Student", that way Purchaser could be an Individual seat purchase in which case the Student and Purchaser are same person automatically, until they give it away, or a team seats Purchaser, where Students start claiming seats. Seat status would be claimed / unclaimed. Coupons is a stupid word here, get rid of it. It's a Seat, and it's claimed or unclaimed. A team seat purchaser can enter email addresses and those indivduals get an email with a link to access the seat and enter any further info needed.

**Q6 - How do the eligibility rules apply to Seats?** Each Seat judged on its own; Refund = per-seat price × eligible seats.
➡️ Yes.

**Scott:**
> Why don't we do it this way: If a team purchase is say 10 seats, and only 5 are claimed, the rest remain open for eternity, and the purchaser can assign them to other people later. If they want those seats cancelled, they must actively go in and cancel them. If a Seat is claimed and that Student is eligible to cancel per the cancellation criteria same as individual Seats, then that seat can be vacated by the Student or Purchaser, left open until the Purchaser actively cancels the seat.

**Q7 - Should the Team Purchaser pick seats from a seat map up front, rather than asking for a number?**
➡️ Yes; no waiting state.

**Scott:**
> Purchaser of team seats should get a seat map, and click on each, enter email addresses. Seat then show claimed. Should be certain same email address can't be assigned to two seats in the same course.

**Q8 - What is a Seat Holder's "cancel" called, and how can it end?** Proposed "Seat Release Request", Awaiting Team Purchaser, three outcomes, 14-day expiry.
➡️ Yes.

**Scott:**
> A Seat Holder should see "Return Seat to your Team" and in finer print, "your seat is eligible to be used by another team member", and the Purchaser would be notified of the seat being vacated and available for re-use. Please add this to an IDEAS.md file: "Because these are digital classes, there is little lost by us if a Student or Purchaser decides part way in to cancel entirely, so we need to review later how to not be jerks about refunds and so on, and we should create a simple state machine design that manages options and transitions for the Seat Lifecycle."

**Q9 - One open request per Purchase?** Separate Purchases, including the same course bought twice, are independent.
➡️ Yes.

**Scott:**
> Purchases cannot be independent for a team seat Purchase of more than 1 Seat. Why? Because we want the Purchaser to have one seat map that shows all available seats for the given course, no matter how many adds and cancellations of seats they make. We don't want them going to separate UIs, one for each purchase, for the same course.

**Q10 - When does access end, and what happens after a Failed Refund?**
➡️ Access ends at the request; retry up to 3 times, then alert the team; access not restored.

**Scott:**
> Access ends for any Student who cancels their Seat, at the moment of cancellation. The Seat is 'deleted' (but kept historically) if it is an Individual Purchaser, or it is put back in a Team Purchaser's pool of seats for that course and it changed from claimed to unclaimed. (I see you already said some of this, oh well.)

**Q11 - Require a reason, with a category?**
➡️ Yes, a category plus optional free text.

**Scott:**
> Let it be free form text for now, we will keep free form always, but add checkboxes as we review the reasons and find soome common ones.

*Written after Round 3:* `GLOSSARY.md` rewritten (Refund Request removed; Purchaser, Seat, Student, Claimed/Unclaimed Seat, Intro Module added); `IDEAS.md` created.

---

## Round 4

Fact found: no cohort concept in the code; every course is always available.

**Q1 - Is this the exact Refund Policy?** Within 30 days of window start; fewer than 4 Completed; fewer than 8 Started lessons outside the Intro Module. Unclaimed Seats only need the window.
➡️ Yes.

**Scott:** Yes.

**Q2 - When does the window start for a cohort course?**
➡️ The later of the cohort start and the purchase date.

**Scott:**
> Let's change this: Any course, cohort or not, can be cancelled at any time, until #2 and/or #3 are reached. This may remove hesitation when buying a course. A 30 day window is stupid in our case, but we want to close our books on purchases at some point, so let's make the window 365 days from date of when a course starts / becomes available, not when it is purchased. People need the ability to do the intro module and up to 4/8 videos to decide if it's helpful for them. I have no business sense.

**Q3 - Name cohort courses now, but build them later?**
➡️ Yes; add Cohort Course and Self-Paced Course to the glossary; building to FUTURE.md.

**Scott:**
> Yes, add those to the Glossary, cohorts are the same as self-paced if the Student wants, but cohorts also have a live office hours from course first opening until our defined and of cohort. A cohort course turns into a self-paced-only course after the live portion ends.

**Q4 - When does a lesson count as "completed"?**
➡️ At 90% watched.

**Scott:**
> It counts as completely before the end of the video only if we can automatically identify the outro start in each video. Otherwise, it is to the end of the video. Risk is that many stop when outro starts because they move on to the next, but they'll hit the 8 partial completed videos in that case.

**Q5 - Which rules apply to returning a Seat?** Gate returning by the watch rules only, not the window.
➡️ Yes.

**Scott:**
> Miscommunication: A cancel on an Individually purchased seat equals a refund and seat gets deleted. A cancel on a Team purchased seat, means the seat gets marked unclaimed, the Purchaser of the Team seats gets to re-assign or request cancellation of the Seat, and they can put that same person back into a Seat if they choose to. If an Individual Purchaser has given their Seat to someone else, and that other person cancels the seat, the Individual Purchaser is refunded, period, and no reassignments after the first one, no reclaiming the seat.

**Q6 - Who can do what to a Seat?** Give (Purchaser), Return (Student or Purchaser), Cancel (Purchaser only, creates a Refund).
➡️ Yes; Unclaimed ↔ Claimed, Refunded is final.

**Scott:**
> Yes, the table is correct, except for Individual Purchaser, the Seat they purchase is automatically Claimed by them, they get to give it to someone else Once, the Seat remains claimed, but by the other person. A cancellation can be made by the Individual Purchaser or the Student they gave the Seat to, but Refunds always go back to the original Purchaser / their form of payment.

**Q7 - Is a Seat claimed as soon as an email is entered, or only once the person accepts?**
➡️ As soon as the email is entered.

**Scott:**
> Seat is claimed once an email address is associated with it.

**Q8 - Does "one email, one Seat per course" apply across Purchasers?**
➡️ Yes; block the team Seat assignment.

**Scott:**
> If a Team Purchaser adds an email to a Team Seat, and that email address is associated with an Individual Purchased Seat, the latter triggers a cancellation and Refund to the Individual Purchaser, period.

**Q9 - Is the seat map a view that keeps each Seat's own purchase details?**
➡️ Yes.

**Scott:**
> Team Seats purchased at different times but for the same course, maintain their 365 day window on a per-Seat basis. Seats are filled by Seat-with-nearest-expiration-date, and we check to see if a Seat will expire before a course is set to become available, or during however we expect that course to last. If the Seat will expire before the course and its duration, we silently extend the Seat's expiration to be after the course is available, and it's defined duration to complete. Add to FUTURE.md to investigate having "Seat Blocks" where each Block is a separate purchase, but when viewed, shows each block's separate expiration date.

**Q10 - One Refund per purchase covered by a cancellation?**
➡️ Yes.

**Scott:**
> We track which purchase event paid for which seat, and we return the soonest to expire seats first, in order. If X seats are cancelled, and they cross multiple purchase events, we count up the number of seats in one block that are cancelled, and refund those, then do the next block, etc.

**Q11 - Retry Failed Refunds, then alert the team?**
➡️ Yes.

**Scott:** Yes.

**Q12 - Record two ADRs?** (a) No team approval; (b) No cancellation record.
➡️ Yes.

**Scott:**
> Yes, but returing a seat only works for Team Purchased Seats, they go back to their block as unclaimed. Individually Purchase Seats are Refunded to the Purchaser, even if the Purchaser gave that seat to someone else, and that someone else cancelled.

*Written after Round 4:* `GLOSSARY.md` extended (courses, lessons, Individual/Team Purchase, Returned/Refunded Seat); ADRs 0001 and 0002; `FUTURE.md` (cohorts, outro detection, Seat Blocks).

---

## Round 5

**Q1 - Where does the 365-day window start?** Counting from the course opening would make late buyers ineligible at once.
➡️ 365 days after the later of purchase and course opening, extended to cover the course's expected duration.

**Scott:** Yes. *(Superseded by Q2.)*

**Q2 - Is "expiration" only the Refund Deadline?**
➡️ Yes; Seats never expire, only the Refund Deadline passes.

**Scott:**
> You really are OCD. Ok, I really want seats to be open for eternity, but I also don't want a refund being requested 10 years later. Why not? Well, I guess it really doesn't matter if that happens, inflation has eroded the value of the payment, so I'm effectively repaying 'less' than the original payment. Let's simplify. All seats are available for eternity, until the 4/8 video rule is hit for a Seat. That may require failed refunds if credit cards have changed, etc., so put that issues into FUTURE.md.

**Q3 - Do watch counts belong to the Seat or to the Student?** Loopholes: Transfer after watching; returning and swapping on a team.
➡️ The Seat.

**Scott:**
> Great catch! Once a Student has met the ineligibility for cancellation rule, they are LOCKED to the Seat they are in, it cannot be vacated or reassigned, or refunded, it is moved into a "Used Seats" list, and remains there. If they do 3 videos, then returns the seat, and is later assigned a seat in that same course, the 3 video completion follows her. So the content reviewed is tied to the Student, even if they play musical chairs. This does mean that a Student can have multiple emails, and a Team Purchaser could put that same student into an unclaimed seat with that different email address, but they'd need to keep changing their email address to play that game. Put that issue in FUTURE.md.

**Q4 - Can a Team Student return a Seat after passing the watch limits?**
➡️ (b), only while within the limits.

**Scott:**
> No, once passed the watch limit for a Student, the Seat is no longer able to be Unclaimed and remains in the system forever for that Student.

**Q5 - A team Seat replaces an Individual Seat that's already ineligible?**
➡️ Refund her anyway.

**Scott:**
> Refund her, the Seat she claims on the Team is immediately locked to her and cannot go Unclaimed.

**Q6 - When a Team Purchaser cancels Seats, which ones go?**
➡️ Unclaimed only, nearest deadline first; return claimed Seats first.

**Scott:**
> Teams seats show up as Claimed or Unclaimed, and a third option, Locked to Student. If a team purchaser requests refunds of 5 out of 10 seats, but 6 are claimed, the UI shows the error. They then can 'Unseat' someone, but if all the rest of the Seats are Locked, then they can only cancel seats that are eligible.

**Q7 - Call it "Transfer" when an Individual Purchaser gives their Seat away?**
➡️ Yes.

**Scott:** Yes.

**Q8 - How should the code changes be split?** (a) terminology fix PR; (b) PRD issue for the new Seat and Refund model.
➡️ Both, (a) first.

**Scott:** Do both.

*Written after Round 5:* `GLOSSARY.md` (Locked Seat, Transfer, Student-owned watch counts, time-free Refund Policy, the Individual-to-Team exception); `FUTURE.md` (refunds to stale cards, multiple-email and Transfer loopholes).

---

## Round 6

**Q1 - Which Seats go first, now that there are no deadlines?** Give out Unclaimed Seats from the oldest purchase; cancel from the newest.
➡️ Yes.

**Scott:**
> Yes, caveat is if they have 10 seats in first purchase and 10 seats in second purchase, and they want to cancel 12 seats, 10 seats must be refunded from the newest purchase, and 2 from the oldest.

**Q2 - One domain term for "Unseat" and "Return"?**
➡️ Yes: Returned Seat.

**Scott:**
> Ignore Unseat, that's me being non-deterministic again.

---

## Shared understanding (confirmed)

1. **Terms**: Refund is the root of every name in the code. "Cancel" appears only in UI text. Chargeback, Reversal and Cancellation are banned as domain terms. Coupon becomes Seat.
2. **Seats**: Unclaimed → Claimed → Locked (permanent), or → Refunded. Only Team Purchase Seats can be Returned (by the Student or the Purchaser). Individual Seats can be Transferred once, and cancelling one always refunds the Purchaser. Seats never expire.
3. **Refund Policy**: fewer than 4 Completed and fewer than 8 Started Lessons outside the free Intro Module, counted per Student per course. No time limit. Judged by the policy version in force at purchase. No team approval, and a Refund exists only if an eligible cancellation happened.
4. **Refund**: `in_progress → completed | failed`, with automatic retries, then an alert to the team. Each Refund covers the Seats from one purchase and stores a free-text reason. Seats are given out from the oldest purchase first and cancelled from the newest first (e.g. cancelling 12 of 10 + 10 takes all 10 from the newest, then 2 from the oldest).
5. **Work order**: (a) a small terminology-only PR. (b) A PRD issue on `origin` for the new Seat and Refund model, split into implementation issues.
