---
title: "Bumara - Developer Onboarding"
description: "Welcome to the team! This guide will help you get up to speed."
---

Welcome to the team! This guide will help you get up to speed.

---

## Your First Week

### Day 1: Setup & Orientation

**Morning (9:00 AM - 12:00 PM):**

- [ ] 9:00 - **Welcome Meeting** (1 hour, whole team)
  - Introductions
  - Product overview
  - Tech stack walkthrough
  - Team process
  
- [ ] 10:30 - **1:1 Setup Session** (1 hour, with CTO)
  - Set up local environment
  - Get all access credentials
  - Clone repo and run app
  - Make first commit

**Afternoon (1:00 PM - 5:00 PM):**

- [ ] 1:00 - **Architecture Deep Dive** (1 hour, whole team)
  - System architecture
  - Code walkthrough
  - Database schema
  - Common patterns
  
- [ ] 2:30 - **Pick First Task** (30 min)
  - Browse GitHub Project board
  - Pick a "good first issue"
  - Ask questions about the task
  
- [ ] 3:00 - **Start Coding!**
  - Work on first task
  - Ask questions in Teams
  - Pair with another dev if helpful

### Day 2-5: First Tasks

**Daily Routine:**

- 9:10 AM - Daily standup (15 min)
- 9:25 AM - Check in with CTO (15 min)
- Rest of day - Code!

**Goals:**

- Complete 1-2 small tasks
- Get comfortable with codebase
- Learn the development workflow
- Ask lots of questions

### Week 2-4: Ramp Up

- Join sprint planning
- Take on medium-sized tasks
- Do code reviews
- Contribute to discussions
- Start working more independently

---

## Team Structure

**CTO (NSANGU):**

- Product decisions
- Technical decisions
- Code reviews
- Unblocking

**Senior Developer (RAY):**

- Code reviews
- Unblocking

**Developers (Everyone):**

- Write code
- Review code
- Help each other
- Contribute ideas
- Own features

**We're all equal contributors. There's no hierarchy in the dev team.**

---

## How We Work (Scrum Process)

### Sprint Structure

**Sprint Length:** 2 weeks

**Sprint Cycle:**

- Week 1 Mon: Sprint Planning
- Mon-Fri: Daily Standup + Work
- Week 2 Fri: Sprint Review + Retrospective

### Meetings

#### Daily Standup (15 minutes, 9:00 AM)

**Format:**
Each person shares (2-3 minutes):

1. What I did yesterday
2. What I'm doing today
3. Any blockers

**Rules:**

- Stand up (keeps it short)
- No problem-solving (take offline)
- Peer-to-peer (not reporting to CTO)
- Exactly 15 minutes

#### Sprint Planning (2 hours, start of sprint)

**What we do:**

- Review sprint goal
- Look at user stories
- Estimate effort
- Commit to what we can complete
- Assign stories

#### Sprint Review (1 hour, end of sprint)

**What we do:**

- Demo completed work
- Get feedback
- Discuss what's next

**Every developer:**

- Demo what you built
- Show it working (not the code)
- Explain decisions you made

#### Sprint Retrospective (45 minutes, end of sprint)

**What we do:**

- What went well?
- What didn't go well?
- What will we try differently?

**This is the most important meeting - we improve here.**

---

## Development Workflow

### 1. Pick a Task

1. Go to GitHub Project Board
2. Look in "Ready" column
3. Pick a task (start with small ones)
4. Assign yourself
5. Move to "In Progress"

### 2. Create Branch

```bash
# Format: type/issue-number-brief-description
git checkout -b feature/123-add-product-list
```

### 3. Code

## Before you start

Read the user story
Check acceptance criteria
Ask questions if unclear

While coding:

Write clean, readable code
Add comments for complex logic
Write tests
Test manually
Commit often

### 6. Code Review

What happens:

Another dev reviews your code
They leave comments
You discuss and make changes
They approve
You merge

As a reviewer:

Be kind and constructive
Ask questions
Suggest improvements
Approve if looks good

As an author:

Respond to all comments
Make requested changes
Explain your decisions
Don't take it personally

Review within 4 hours - keep things moving!

### 7. Merge & Deploy

After approval:
bash# Merge to main

# GitHub Actions will deploy to staging automatically

Check staging: Make sure it works
Move issue to "Done"

Code Standards
TypeScript

Use TypeScript strict mode
No any types (use unknown if needed)
Define interfaces for data structures
Use type guards

### Resources

Documentation

Product: docs/PRODUCT/
Architecture: docs/ARCHITECTURE/
This guide: docs/ONBOARDING.md

External Docs

Next.js Docs
Hono Docs
Clerk Docs
Drizzle Docs
Tailwind Docs (if using)

Communication

Daily questions: #dev-team channel on Teams
Urgent help: DM @[CTO name]
Ideas/suggestions: #product-ideas channel
Bugs: Create GitHub issue
