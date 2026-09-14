# AgentSpec — StudySync AI

Team:Clustora

Department: B.Tech RPT & IT

Submitted: 15 September 2026



## 1. The setting

**Who exactly**: A second-year engineering student with an upcoming university exam and several topics to cover.

**What they do today**: A few days before the exam, the student looks at the syllabus, estimates how much time they have each day, and makes a study timetable on their own. When their available time changes or they fall behind on a topic, they manually rearrange the remaining schedule.

**Why that is hard**: The student often overestimates how much they can finish in a day and creates a plan that becomes unrealistic after one missed session or difficult topic. They need a plan that can be discussed, revised, and adjusted around their actual constraints instead of starting from scratch every time.

## 2. The problem this solves

A second-year engineering student has four days before a Thermodynamics exam and plans to study three hours every evening, dividing the syllabus equally across the four days. On the second day, they spend much longer than expected on Entropy and complete only half of what was planned. Instead of adjusting the remaining days, they continue following the original timetable and reach the final day with several important topics still untouched. The student then has to choose what to skip, studies the remaining topics without enough time, and goes into the exam underprepared. The problem is not that the student had no study plan — it is that the plan had no way to respond when reality changed.

## 3. What we are building

**Input**: One student's upcoming exam topic list, the number of days remaining, available study hours, and any changes or constraints they want to make to the plan.

**Output**: A day-by-day study plan that the student can review, reject, or modify, with the final approved plan saved so it can be updated when the student returns.

Never, however much a user wants it: It does not track actual study time, send reminders, connect to calendars or LMS platforms, automatically fetch syllabi, or create plans for multiple exams at once.

**Why this is agentic, in our own words**: The agent maintains the student's plan and revision history between encounters, uses separate steps to draft and review the plan, waits for the student when approval or changes are needed, and sends the plan back for revision when the student rejects it. The number of revisions is decided by the interaction and bounded by a revision limit rather than following one fixed generation.

## 4. A complete walkthrough

A complete walkthrough
**Student**: Arjun, second-year engineering student

**Exam**: Thermodynamics

**Days remaining**: 4

**Available study time**: 3 hours per day

**Input**

Arjun starts a new StudySync session and enters:

"I have my Thermodynamics exam in 4 days. I can study for 3 hours each day. I need to cover First Law, Second Law, Entropy, Thermodynamic Cycles, and Refrigeration. I find Entropy difficult and want more time for it."

The agent converts this into a structured plan request: 


```json
{
  "student": "Arjun",
  "exam": "Thermodynamics",
  "days_remaining": 4,
  "daily_hours": 3,
  "topics": ["First Law", "Second Law", "Entropy", "Thermodynamic Cycles", "Refrigeration"],
  "priority_topic": "Entropy"
}
```

**Step 1** — Draft
The planning step uses the available hours, number of days, topic list, and Arjun's stated difficulty with Entropy. Because Entropy is flagged as the priority topic, the draft allocates it extra time relative to the other topics — a full day alone, plus a 30-minute buffer pulled from Refrigeration's allocation on Day 4

```json
{
  "kind": "study_plan",
  "attempt": 1,
  "student": "Arjun",
  "exam": "Thermodynamics",
  "days": [
    { "day": 1, "hours": 3, "topics": ["First Law", "Second Law"] },
    { "day": 2, "hours": 3, "topics": ["Entropy"] },
    { "day": 3, "hours": 3, "topics": ["Thermodynamic Cycles"] },
    { "day": 4, "hours": 3, "topics": ["Refrigeration (2.5h)", "Entropy — extra revision (0.5h)"] }
  ]
}
```

**Step 2** — Check and ask

The agent does not treat its first plan as final. It presents the plan and asks for approval:
"This plan gives Entropy a full day plus a short revision slot on Day 4, since you flagged it as difficult. Do you approve it, or would you like to change any day or time allocation?"
Arjun does not approve it:
"I have only 1.5 hours on Day 2 because of another commitment. Also, I don't want to study First Law and Second Law on the same day."
The agent records the rejection and the constraints that caused it:

```json
{
  "kind": "plan_feedback",
  "attempt": 1,
  "approved": false,
  "constraints": [
    "Day 2 has only 1.5 available hours",
    "First Law and Second Law cannot be scheduled on the same day"
  ]
}
```

The run moves backwards from review to revision instead of starting a new conversation.

**Step 3** — Revise

The revision step reads the previous plan and Arjun's feedback rather than generating an independent timetable. It must preserve:
No day exceeds Arjun's available hours.

First Law and Second Law are on different days.

Entropy keeps its priority weighting.

All five topics still receive study time.

The final day retains revision time.

```json
{
  "kind": "study_plan",
  "attempt": 2,
  "student": "Arjun",
  "exam": "Thermodynamics",
  "days": [
    { "day": 1, "hours": 3, "topics": ["First Law"] },
    { "day": 2, "hours": 1.5, "topics": ["Entropy"] },
    { "day": 3, "hours": 3, "topics": ["Second Law", "Thermodynamic Cycles"] },
    { "day": 4, "hours": 3, "topics": ["Refrigeration (2.5h)", "Entropy — extra revision (0.5h)"] }
  ]
}
```

The agent asks again:

"The revised plan separates First Law and Second Law, limits Day 2 to 1.5 hours, and still gives Entropy extra time on Day 4 since it's your priority topic. Do you approve this plan?"

Arjun raises one more objection:

"Close — but move that extra Entropy slot to Day 3 instead, since Day 4 is already my heaviest day."

The agent reads this as a targeted edit, not a rejection of the whole plan, and moves the 0.5-hour Entropy block accordingly without touching anything else:

```json
{
  "kind": "plan_feedback",
  "attempt": 2,
  "approved": false,
  "constraints": ["Move Entropy extra-revision slot from Day 4 to Day 3"]
}
```

```json
{
  "kind": "study_plan",
  "attempt": 3,
  "student": "Arjun",
  "exam": "Thermodynamics",
  "days": [
    { "day": 1, "hours": 3, "topics": ["First Law"] },
    { "day": 2, "hours": 1.5, "topics": ["Entropy"] },
    { "day": 3, "hours": 3, "topics": ["Second Law", "Thermodynamic Cycles", "Entropy — extra revision (0.5h)"] },
    { "day": 4, "hours": 3, "topics": ["Refrigeration", "Revision"] }
  ]
}
```

"Yes, I approve this plan."

```json
{
  "kind": "approval",
  "student": "Arjun",
  "approved_attempt": 3,
  "approved_at": "2026-09-15T09:20:00",
  "status": "active"
}
```

**Step 4** — Second encounter

The following day, Arjun returns to StudySync. Instead of asking him to re-enter the exam, topics, and plan, the system loads the saved state and opens with it:

"Welcome back. Your Thermodynamics plan has First Law completed on Day 1 and Entropy scheduled for today, Day 2 — 1.5 hours, since you flagged it as your hardest topic. How did it go?"

Arjun reports:

"I completed First Law yesterday, but I could only study Entropy for 30 minutes today. I still haven't started Second Law."

```json
{
  "kind": "progress",
  "student": "Arjun",
  "exam": "Thermodynamics",
  "completed": ["First Law"],
  "partially_completed": [{ "topic": "Entropy", "completed_hours": 0.5 }],
  "not_started": ["Second Law", "Thermodynamic Cycles", "Refrigeration"]
}
```

**Step 5** — Adapt the remaining plan

The agent compares the saved approved plan (attempt 3) against the new progress record. It does not reschedule First Law, and it does not generate a new four-day plan — it changes only what's unfinished:

```json
{
  "kind": "plan_update",
  "student": "Arjun",
  "based_on_attempt": 3,
  "remaining_days": 2,
  "changes": [
    "First Law removed — already completed",
    "Entropy carried forward — only 0.5h of 1.5h done",
    "Entropy extra-revision slot on Day 3 kept, since the topic is still behind",
    "Second Law, Thermodynamic Cycles, Refrigeration retained",
    "Final revision retained on Day 4"
  ],
  "updated_days": [
    { "day": 3, "hours": 3, "topics": ["Finish Entropy (1h)", "Second Law", "Entropy — extra revision (0.5h)"] },
    { "day": 4, "hours": 3, "topics": ["Thermodynamic Cycles", "Refrigeration", "Final Revision"] }
  ]
}
```


The agent presents the update:

"Your Entropy session was only partially done, so I carried the remaining hour into Day 3 along with the extra revision slot you asked to move there. First Law is off the plan since it's complete. Does this work?"
Arjun can approve it or send it back again, using the same revision path as Steps 2 and 3.
The important difference from a fresh conversation is that StudySync knows what Arjun originally planned, what he negotiated, what he approved, what he completed, and what remains — and it treats "Entropy is hard" as a standing constraint that keeps shaping the plan across both encounters, not a note that was read once and forgotten.


## 5. Who is doing the thinking

| step | the agent does it | the human does it | what the human loses if the agent does it |
|---|---|---|---|
| Understand the student's topics, available hours and constraints | Extracts and structures the information provided by the student | Provides the actual exam topics, available time and personal constraints | The plan could be based on assumptions instead of the student's real situation |
| Create the first study plan | Allocates topics across the available days while following the stated constraints | Reviews the proposed plan and identifies whether any allocation conflicts with their preferences or priorities | Nothing essential; this is a repeatable planning task |
| Review the plan against the student's situation | Checks the plan against the stated constraints and presents it for review | Decides whether the plan actually fits their schedule and states what needs to change | Control over their own schedule, priorities and preferences |
| Revise the plan after feedback | Reallocates the remaining topics while preserving the student's stated constraints | Specifies what should change when the plan does not fit | The ability to decide what should be prioritised, avoided or changed |
| Approve the plan | Records the decision and saves the approved plan | Explicitly approves or rejects the revised plan | Final control over what they actually intend to follow |
| Adapt the plan after the student returns | Compares the saved plan with the student's reported progress and proposes changes to the remaining plan | Reports what was actually completed, what was partially completed and what remains; approves or rejects the updated plan | The agent would be assuming what the student completed and deciding what they should do next without confirmation |

The question it asks, and who answers it:
After generating or revising a plan, the agent asks the student: "Does this plan work for you, or would you like to change any day, topic, or time allocation?" The student who owns the plan answers. During a second encounter, the agent also asks: "What did you complete, what was only partially completed, and what did you not start?"
What happens if nobody answers, and how the output shows that:
The plan remains in a waiting-for-approval state and is not marked as approved. The stored record shows "waiting — student response required" rather than silently treating the plan as accepted. When the student returns, the system presents the saved draft and resumes from the waiting state. If no response is ever given, the output remains explicitly marked "not approved — waiting for student response."

## 6. The state machine
 
### How StudySync works

```text
Student gives exam details
        |
        v
Agent understands the student's
topics, time and difficulties
        |
        v
Agent creates a study plan
        |
        v
Student reviews the plan
        |
        +----------------------+
        |                      |
     "Yes"                  "Change it"
        |                      |
        v                      v
     Plan saved          Agent revises plan
        |                      |
        |                      |
        |                +-----+-----+
        |                |           |
        |             Revised    More changes
        |             plan ready      needed
        |                |           |
        |                +-----------+
        |                      |
        |                      v
        |               Student reviews
        |                  again
        |
        v
Student follows the plan
        |
        | Student comes back later
        v
Agent loads the saved plan
        |
        v
Student tells what was
completed / not completed
        |
        v
Agent adjusts the remaining plan
        |
        v
Student reviews the updated plan
        |
        +---- Yes ----> Plan continues
        |
        +---- Change ----> Agent revises again


If the student does not respond
        |
        v
     Waiting
        |
        v
Student returns
        |
        v
System continues from
where it stopped
```

### States


| State            | Type     | What happens here                                                                               | What moves it forward                         |
| ---------------- | -------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------- |
| `student_input`  | Active   | Student gives the exam, topics, available time, difficult topics and other preferences.         | Student submits the details.                  |
| `understanding`  | Active   | StudySync organises what the student has said so it can plan around the real situation.         | The information needed for planning is clear. |
| `planning`       | Active   | StudySync creates a study plan based on the student's information.                              | A plan is ready to show the student.          |
| `student_review` | Waiting  | The student looks at the plan and decides whether it works.                                     | Student approves it or asks for changes.      |
| `revising`       | Active   | StudySync changes the plan using the student's feedback instead of starting again from zero.    | A revised plan is ready.                      |
| `active_plan`    | Active   | The approved plan is saved and becomes the student's current plan.                              | Student returns later or completes the plan.  |
| `progress_check` | Waiting  | When the student returns, StudySync asks what was actually completed and what is still pending. | Student reports their progress.               |
| `waiting`        | Waiting  | StudySync has asked the student something but has not received an answer yet.                   | Student returns and answers.                  |
| `completed`      | Finished | All planned topics are completed.                                                               | Nothing. This is the end of the plan.         |
| `stopped`        | Finished | The system reaches its revision or usage limit before reaching an approved plan.                | Nothing. This is the end of the run.          |

### What can send the work backwards

| Situation                                                      | What happens                                         |
| -------------------------------------------------------------- | ---------------------------------------------------- |
| Student does not like the plan                                 | `student_review → revising → planning`               |
| Student gives new constraints                                  | `student_review → revising → planning`               |
| Student returns with different progress from the original plan | `progress_check → planning`                          |
| Student has not answered a question                            | The run stays in `waiting` until the student returns |

What the run decides
StudySync decides whether the current plan needs to be changed based on the student's feedback and progress. If the information is not enough, it waits for the student instead of guessing.
The system also decides when another revision is no longer useful and stops when the limits are reached.
Spend limit
The run has a maximum of 12 model calls. Retries are also counted.
If the limit is reached, the system stops instead of continuing to make more calls.
Revision limit
The plan can be revised a maximum of 3 times per encounter.
The revision count and model-call count are separate. A failed model call does not count as a plan revision.
Important rule
StudySync can create and revise a plan, but it cannot approve its own plan.
Only the student can approve the plan and move it into the active_plan state.

### Why I think this is stronger

The important thing is that a judge can look at it and immediately understand:

**Student tells → AI understands → AI plans → Student decides → AI revises if needed → Plan is saved → Student comes back → AI adapts using actual progress.**

That is much more human-readable than `drafting → pending_approval → revision → active → collecting_progress`.

But you still retain the **agentic parts** the organisers are specifically looking for: **memory between encounters, human-in-the-loop, backward movement, waiting, revision limits, and spend limits.**

## 7. The data model


## 8. Step-by-step contracts


## 9. The second encounter

The next day, Arjun comes back to finish his plan. Here's the difference memory makes.

**Without memory:** Arjun would have to start over — re-explain the exam, list all five topics again, say Entropy is hard again, and re-negotiate the Day 2 time limit he already fought for. Every earlier conversation would be wasted effort.

**With memory:** StudySync opens the session already knowing his approved plan — what's scheduled, what he flagged as hard, what he already decided. Three things become possible because of that:

1. **It can tell if he's behind, not just what he did.** The plan said 1.5 hours of Entropy on Day 2. He reports 30 minutes. The system doesn't just log "30 minutes" — it knows that's an hour short, because it knows what the target was.

2. **It keeps decisions he already made.** Arjun didn't just say Entropy was hard — he specifically asked to move its extra revision slot to Day 3 instead of Day 4. That stays in place. He doesn't have to ask twice.

3. **It leaves finished work alone.** First Law is done, so it disappears from what's left to do — no re-asking, no risk of accidentally scheduling it again.

**The simple test:** if you deleted the stored plan and made Arjun explain everything from scratch, would the next step still work correctly? No — it needs the exact numbers and decisions from before to know what's changed. That's how you know the memory is actually being used, not just stored for show.

## 10. Files and responsibilities


## 11. What this deliberately does not do

**1. We're not letting it decide if Arjun should even be studying this way.** The system takes his exam, his topics, and his hours as a given the second he opens the app. It never asks "is this actually a good plan for your life right now?" We talked about having it push back if the numbers looked crazy — five hard topics, four hours total — but pulled back from that. That's not scheduling anymore, that's judging someone's choices with information we don't actually have. Arjun gets to decide what he's doing. We just help him do it.

**2. It never approves its own plan, no matter how many times he rejects it.** We genuinely considered a shortcut here — if he says no three times in a row, just lock in the last version so the demo doesn't stall out. We killed that idea. The second the agent can wave its own plan through, "approved" doesn't mean anything anymore, and the whole point of asking was theater. Slower and honest beats fast and fake.

**3. It only handles one exam at a time.** Real students juggle two or three exams in the same week — we know that. We wanted to build for it. We didn't, because splitting hours between subjects is a genuinely harder problem than we had time to solve properly, and a good plan for one subject beats a shaky plan for two.

**4. It won't check in on Arjun if he goes quiet.** If he doesn't open the app for two days before his exam, the system doesn't notice or nudge him. We liked the idea of a gentle "hey, your exam's close and we haven't heard from you" — but that means the agent starts reaching out on its own instead of just answering when asked, and that's a different, bigger thing to get right than what we set out to build.

**5. It takes his word for it.** If Arjun says he finished First Law, the system believes him. No quiz, no check on whether he actually understood it. We thought about adding one, and decided that's a tutoring problem, not a planning problem — and trying to solve both would probably mean we do neither one well.

## 12. Build order

| Phase | What lands                                                                                                                               | Hours |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------- | ----: |
| 1     | Build the complete flow with fixed sample responses: student input → understanding → study plan → student review → revision → approval.  |     4 |
|       | **Cut line:** We can demonstrate the complete planning and revision flow, including the student approval step, even without live AI.     |       |
| 2     | Add real AI responses for understanding, planning and revision. Store the approved plan so it can be loaded later.                       |     6 |
|       | **Cut line:** A real student input can produce a personalised plan, receive feedback, revise it and save the approved plan.              |       |
| 3     | Add the second encounter: load the saved plan, collect the student's actual progress, and adjust only the remaining study plan.          |     5 |
|       | **Cut line:** We can demonstrate that StudySync remembers the previous plan and changes it based on what the student actually completed. |       |
| 4     | Add waiting/resume behaviour, revision and model-call limits, error handling, and clean up the demo interface.                           |     3 |
|       | **Cut line:** The complete agent can pause for the student, resume later, and safely stop when its limits are reached.                   |       |

### Where the hours will actually go

Most of our time will go into testing whether the generated study plans are actually useful and whether the agent responds correctly to different student situations.

We will test cases such as limited study time, difficult topics, changing schedules, unfinished topics and conflicting constraints. We expect prompt refinement and checking the quality of these outputs to take more time than writing the basic code.

We will build the complete flow with fixed responses first and save useful model responses for replay during testing. This gives us a working path early and reduces the risk of having nothing to demonstrate if the live model behaves unexpectedly.

## 13. The demo


## 14. How this grows



## 15. What you are least sure about

1. **Whether the generated study plan is actually realistic.**  
   We are not sure whether the agent will consistently create plans that fit the student's available hours while still giving enough time to difficult topics. We will test the same student input multiple times and check whether the plans follow all the stated constraints.

2. **Whether the agent makes useful changes after student feedback.**  
   We are not sure whether the revision step will genuinely improve the plan or simply rearrange the same topics. We will test different types of feedback, such as reduced study time, topic priorities and schedule changes, and compare the revised plan with the original.

3. **Whether the second encounter correctly uses the student's actual progress.**  
   We are not sure whether the system will reliably remember the previous plan and adjust only the unfinished work instead of creating a completely new plan. We will save a plan, return with different progress scenarios, and check whether the updated plan reflects exactly what the student completed. 
## 16. Claims to verify

