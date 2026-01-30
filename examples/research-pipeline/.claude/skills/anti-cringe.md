---
name: anti-cringe
type: domain
version: v1.0
description: "AI-content detection and quality filter for generated text"
---

# Anti-Cringe Filter

## Purpose

Detect and eliminate AI-typical writing patterns that reduce content quality and authenticity. Ensure generated text reads naturally.

## Patterns to Avoid

### Category 1: Filler Phrases

**Banned:**
- "It's important to note that..."
- "It's worth mentioning that..."
- "Interestingly enough..."
- "As we can see..."
- "In today's fast-paced world..."
- "In the ever-evolving landscape of..."
- "When it comes to..."

**Fix:** Delete the phrase, start with the actual content.

### Category 2: Weak Transitions

**Banned:**
- "Moving on to..."
- "Let's dive into..."
- "Now let's explore..."
- "That being said..."
- "With that in mind..."

**Fix:** Use direct transitions or none at all.

### Category 3: Excessive Hedging

**Banned:**
- "It could potentially..."
- "This might possibly..."
- "It seems like it may..."
- "One could argue that..."

**Fix:** Be direct. State the claim with appropriate confidence.

### Category 4: Hollow Conclusions

**Banned:**
- "In conclusion..."
- "To sum up..."
- "All in all..."
- "At the end of the day..."
- "In summary, we have seen that..."

**Fix:** Just state the conclusion.

### Category 5: False Profundity

**Banned:**
- "This is a game-changer"
- "Revolutionary approach"
- "Paradigm shift"
- "Cutting-edge"
- "Best practices"
- "Synergy"
- "Leverage"

**Fix:** Use specific, concrete descriptions.

### Category 6: Unnecessary Acknowledgment

**Banned:**
- "Great question!"
- "That's a really interesting point"
- "I'm glad you asked"
- "Absolutely!"

**Fix:** Just answer the question.

### Category 7: Repetitive Structure

**Pattern to avoid:**
```
Point 1. Explanation.
Point 2. Explanation.
Point 3. Explanation.
```

**Fix:** Vary sentence structure and paragraph length.

### Category 8: Over-enumeration

**Pattern to avoid:**
- Listing 5+ items when 3 would suffice
- Bullet points for everything
- Numbered lists for non-sequential items

**Fix:** Use prose when appropriate. Be selective.

## Quality Indicators

### Good Writing Signals
- Specific examples over generic statements
- Concrete numbers over vague quantities
- Active voice over passive
- Short sentences mixed with longer ones
- Direct claims with evidence
- Natural paragraph flow

### Warning Signs
- Every paragraph starts the same way
- Heavy use of "This", "It", "There"
- Sentences all similar length
- Abstract language without examples
- Lists within lists

## Procedure

### Step 1: Scan for Banned Phrases
Check against all banned patterns.

### Step 2: Evaluate Structure
- Sentence variety?
- Paragraph flow?
- Appropriate list usage?

### Step 3: Check Substance
- Concrete examples?
- Specific evidence?
- Direct claims?

### Step 4: Revise
For each flagged issue:
1. Delete filler
2. Rewrite weak transitions
3. Strengthen hedged claims
4. Simplify conclusions
5. Replace buzzwords with specifics

## Integration

### With Generators
- Run anti-cringe check before output
- Flag and revise detected patterns
- Prefer substance over style

### With Validators
- Include anti-cringe score in quality metrics
- Flag content with multiple violations

## Examples

### Before (Bad)
```
It's important to note that in today's ever-evolving landscape of
AI development, multi-agent systems have emerged as a game-changing
paradigm. Let's dive into the key aspects of this revolutionary approach.

First and foremost, we need to consider...
```

### After (Good)
```
Multi-agent systems coordinate multiple AI models to solve complex tasks.
Three architectural patterns dominate current implementations:
hierarchical, mesh, and swarm.

The hierarchical pattern...
```
