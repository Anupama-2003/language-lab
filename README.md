

I'll now provide a **comprehensive deep analysis** of every question based on both your documents, followed by a **detailed implementation plan** for each issue.

---

# 🔬 MEGA DEEP ANALYSIS & IMPLEMENTATION PLAN

## Complete Timetable System Overhaul for Indian K-12 Schools

---

# 📊 QUESTION 1: Double/Triple Periods

## Current State Analysis

```
CURRENT STATUS: 85% Complete ⚠️

What EXISTS:
✅ subject_configurations table with:
   - weekly_frequency: 5
   - double_period_allowed: true/false
   - double_period_required: true/false
   - max_doubles_per_week: 2

What WORKS:
✅ Pattern 1+1+1+1+1 (5 singles)
✅ Pattern 2+2+1 (2 doubles + 1 single)
✅ Pattern 1+1+1+2 (3 singles + 1 double)

What's BROKEN/MISSING:
❌ Triple periods (3 consecutive) - flag exists but NO logic
❌ No explicit lesson_pattern column
❌ No min_consecutive_periods column
❌ Algorithm expands doubles but doesn't guarantee consecutiveness
❌ No validation that double periods are ACTUALLY consecutive in output
```

## Root Cause Analysis

```
PROBLEM 1: The scheduler creates "double" entries but may place them
in non-consecutive slots due to greedy algorithm

Example Bug:
  Input: Science Lab needs double period
  Expected: Monday P3-P4 (consecutive)
  Actual: Monday P3 + Monday P6 (NOT consecutive!) ❌

PROBLEM 2: Triple periods have no expansion logic
  Code: if (double_period_allowed) → creates [Double, Single, Single]
  Missing: if (triple_period_allowed) → creates [Triple, Single, Single]

PROBLEM 3: No lesson_pattern column means admin cannot specify
  "I want exactly 2+2+1 pattern for Science"
  System guesses the pattern from boolean flags
```

## Implementation Plan

### Phase 1: Database Changes

```sql
-- MIGRATION: 005_lesson_patterns.sql

-- Step 1: Add lesson_pattern column to subject_configurations
ALTER TABLE subject_configurations
ADD COLUMN lesson_pattern VARCHAR(50) DEFAULT NULL,
-- Examples: "1+1+1+1+1", "2+2+1", "2+1+1+1", "3+2", "2+2"
ADD COLUMN min_consecutive_periods INTEGER DEFAULT 1,
ADD COLUMN max_consecutive_periods INTEGER DEFAULT 2,
ADD COLUMN triple_period_allowed BOOLEAN DEFAULT false,
ADD COLUMN triple_period_required BOOLEAN DEFAULT false,
ADD COLUMN max_triples_per_week INTEGER DEFAULT 0,
ADD COLUMN preferred_slots_for_doubles JSONB DEFAULT NULL;
-- Example: {"preferred_periods": [3,4], "preferred_days": [1,3,5]}

-- Step 2: Add consecutive validation tracking
CREATE TABLE lesson_pattern_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID REFERENCES schools(id),
  name VARCHAR(100) NOT NULL,
  -- e.g., "Science Lab Pattern", "Sports Pattern"
  pattern VARCHAR(50) NOT NULL,
  -- e.g., "2+2+1", "3+2", "1+1+1+2"
  total_periods INTEGER NOT NULL,
  -- Auto-calculated from pattern
  consecutive_blocks JSONB NOT NULL,
  -- e.g., [{"size": 2, "count": 2}, {"size": 1, "count": 1}]
  applicable_subjects UUID[] DEFAULT NULL,
  -- Which subjects can use this pattern
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Step 3: Insert default Indian school patterns
INSERT INTO lesson_pattern_templates (school_id, name, pattern, 
  total_periods, consecutive_blocks) VALUES
('school-uuid', 'All Singles', '1+1+1+1+1', 5, 
  '[{"size":1,"count":5}]'),
('school-uuid', 'One Double Rest Singles', '2+1+1+1', 5, 
  '[{"size":2,"count":1},{"size":1,"count":3}]'),
('school-uuid', 'Two Doubles One Single', '2+2+1', 5, 
  '[{"size":2,"count":2},{"size":1,"count":1}]'),
('school-uuid', 'Lab Pattern', '2+2', 4, 
  '[{"size":2,"count":2}]'),
('school-uuid', 'Sports Pattern', '3+2', 5, 
  '[{"size":3,"count":1},{"size":2,"count":1}]'),
('school-uuid', 'Art/Craft Pattern', '2+1+1', 4, 
  '[{"size":2,"count":1},{"size":1,"count":2}]');

-- Step 4: Add consecutive tracking to timetable_slots
ALTER TABLE timetable_slots
ADD COLUMN is_consecutive_start BOOLEAN DEFAULT false,
ADD COLUMN consecutive_group_id UUID DEFAULT NULL,
ADD COLUMN consecutive_position INTEGER DEFAULT 1;
-- position 1 = first of double/triple, 2 = second, 3 = third
```

### Phase 2: Lesson Pattern Parser

```javascript
// NEW FILE: lesson-pattern.parser.js

class LessonPatternParser {
  
  /**
   * Parse pattern string into blocks
   * Input: "2+2+1"
   * Output: [
   *   { size: 2, index: 0 },
   *   { size: 2, index: 1 },
   *   { size: 1, index: 2 }
   * ]
   */
  static parsePattern(patternString) {
    if (!patternString) return null;
    
    const blocks = patternString.split('+').map((size, index) => ({
      size: parseInt(size),
      index,
      isDouble: parseInt(size) === 2,
      isTriple: parseInt(size) >= 3
    }));
    
    const totalPeriods = blocks.reduce((sum, b) => sum + b.size, 0);
    
    return {
      pattern: patternString,
      blocks,
      totalPeriods,
      doubleCount: blocks.filter(b => b.isDouble).length,
      tripleCount: blocks.filter(b => b.isTriple).length,
      singleCount: blocks.filter(b => b.size === 1).length
    };
  }
  
  /**
   * Expand pattern into schedulable occurrences
   * Input: { pattern: "2+2+1", subjectId: "math" }
   * Output: [
   *   { subjectId: "math", slotsNeeded: 2, blockIndex: 0 },
   *   { subjectId: "math", slotsNeeded: 2, blockIndex: 1 },
   *   { subjectId: "math", slotsNeeded: 1, blockIndex: 2 }
   * ]
   */
  static expandToOccurrences(subject) {
    const parsed = this.parsePattern(
      subject.lessonPattern || 
      this.inferPattern(subject)
    );
    
    if (!parsed) {
      // Fallback: all singles
      return Array(subject.weeklyFrequency).fill({
        ...subject,
        slotsNeeded: 1,
        blockIndex: 0,
        isConsecutiveStart: false
      });
    }
    
    return parsed.blocks.map((block, idx) => ({
      ...subject,
      slotsNeeded: block.size,
      blockIndex: idx,
      isConsecutiveStart: block.size > 1,
      isDouble: block.isDouble,
      isTriple: block.isTriple
    }));
  }
  
  /**
   * Infer pattern from boolean flags (backward compatibility)
   */
  static inferPattern(subject) {
    const freq = subject.weeklyFrequency;
    const maxDoubles = subject.maxDoublesPerWeek || 0;
    const doubleRequired = subject.doublePeriodRequired;
    const tripleAllowed = subject.triplePeriodAllowed;
    
    if (tripleAllowed && freq >= 5) {
      return '3+2'; // Triple + double
    }
    
    if (doubleRequired) {
      const doubles = Math.min(Math.floor(freq / 2), maxDoubles);
      const singles = freq - (doubles * 2);
      const parts = [];
      for (let i = 0; i < doubles; i++) parts.push('2');
      for (let i = 0; i < singles; i++) parts.push('1');
      return parts.join('+');
    }
    
    if (maxDoubles > 0) {
      const singles = freq - (maxDoubles * 2);
      const parts = [];
      for (let i = 0; i < maxDoubles; i++) parts.push('2');
      for (let i = 0; i < singles; i++) parts.push('1');
      return parts.join('+');
    }
    
    // All singles
    return Array(freq).fill('1').join('+');
  }
  
  /**
   * Validate that scheduled slots match the pattern
   */
  static validateConsecutiveness(slots, pattern) {
    const parsed = this.parsePattern(pattern);
    const errors = [];
    
    // Group slots by day
    const dayGroups = {};
    for (const slot of slots) {
      if (!dayGroups[slot.day]) dayGroups[slot.day] = [];
      dayGroups[slot.day].push(slot);
    }
    
    // Check each day's slots for consecutiveness
    for (const [day, daySlots] of Object.entries(dayGroups)) {
      daySlots.sort((a, b) => a.periodSequence - b.periodSequence);
      
      // Find consecutive groups
      let currentGroup = [daySlots[0]];
      for (let i = 1; i < daySlots.length; i++) {
        if (daySlots[i].periodSequence === 
            daySlots[i-1].periodSequence + 1) {
          currentGroup.push(daySlots[i]);
        } else {
          // Gap found - validate current group
          if (currentGroup.length > 1) {
            // This is a consecutive block - verify it matches pattern
            const matchingBlock = parsed.blocks.find(
              b => b.size === currentGroup.length
            );
            if (!matchingBlock) {
              errors.push({
                type: 'INVALID_CONSECUTIVE_SIZE',
                day,
                size: currentGroup.length,
                expected: parsed.blocks.map(b => b.size),
                message: `Day ${day}: Found ${currentGroup.length} ` +
                  `consecutive but pattern expects ` +
                  `${parsed.blocks.map(b=>b.size).join(',')}`
              });
            }
          }
          currentGroup = [daySlots[i]];
        }
      }
    }
    
    return { valid: errors.length === 0, errors };
  }
}

module.exports = LessonPatternParser;
```

### Phase 3: Enhanced Scheduler for Consecutive Periods

```javascript
// MODIFIED: deterministic.scheduler.js

/**
 * Enhanced slot finding for consecutive periods
 * This replaces the simple single-slot finder
 */
findConsecutiveSlots(occurrence, existingSlots, 
  daysPerWeek, gradePeriods) {
  
  const slotsNeeded = occurrence.slotsNeeded;
  const candidates = [];
  
  for (let day = 1; day <= daysPerWeek; day++) {
    // Get teaching periods only (no breaks)
    const teachingPeriods = gradePeriods
      .filter(p => p.is_teaching !== false)
      .sort((a, b) => a.sequence - b.sequence);
    
    // Slide window of size slotsNeeded across periods
    for (let startIdx = 0; 
         startIdx <= teachingPeriods.length - slotsNeeded; 
         startIdx++) {
      
      const windowPeriods = teachingPeriods
        .slice(startIdx, startIdx + slotsNeeded);
      
      // CRITICAL: Verify periods are ACTUALLY consecutive
      // (no break periods in between)
      let isConsecutive = true;
      for (let i = 1; i < windowPeriods.length; i++) {
        const prevSeq = windowPeriods[i-1].sequence;
        const currSeq = windowPeriods[i].sequence;
        
        // Check if there's a break between them
        const breakBetween = gradePeriods.find(p => 
          p.is_break && 
          p.sequence > prevSeq && 
          p.sequence < currSeq
        );
        
        if (breakBetween || (currSeq - prevSeq) !== 1) {
          isConsecutive = false;
          break;
        }
      }
      
      if (!isConsecutive) continue;
      
      // Check ALL periods in window are valid
      let allValid = true;
      for (const period of windowPeriods) {
        if (!this.isSlotValid(
          day, period.id, occurrence, existingSlots
        )) {
          allValid = false;
          break;
        }
      }
      
      if (allValid) {
        // Score this candidate
        const score = this.scoreConsecutiveCandidate(
          day, windowPeriods, occurrence, existingSlots
        );
        
        candidates.push({
          day,
          periods: windowPeriods,
          startPeriodIdx: startIdx,
          score
        });
      }
    }
  }
  
  // Sort by score (lower = better)
  candidates.sort((a, b) => a.score - b.score);
  
  return candidates;
}

/**
 * Score consecutive slot candidates
 * Lower score = better placement
 */
scoreConsecutiveCandidate(day, periods, occurrence, existingSlots) {
  let score = 0;
  
  // 1. Prefer earlier periods for lab/practical subjects
  if (occurrence.requiresLab) {
    score += periods[0].sequence * 2;
    // Labs should be in morning if possible
  }
  
  // 2. Prefer balanced day loads
  const dayLoad = existingSlots.filter(
    s => s.classId === occurrence.classId && 
         s.sectionId === occurrence.sectionId && 
         s.day === day
  ).length;
  score += dayLoad * 3;
  
  // 3. Prefer days where this subject hasn't been scheduled
  const subjectOnDay = existingSlots.filter(
    s => s.classId === occurrence.classId && 
         s.subjectId === occurrence.subjectId && 
         s.day === day
  ).length;
  score += subjectOnDay * 20; // Heavy penalty for same subject same day
  
  // 4. Avoid double periods right after break
  const firstPeriod = periods[0];
  const prevPeriodInAll = this.allPeriods.find(
    p => p.sequence === firstPeriod.sequence - 1
  );
  if (prevPeriodInAll && prevPeriodInAll.is_break) {
    score += 15; // Penalty for double after break
  }
  
  // 5. Preferred slots for doubles (from configuration)
  if (occurrence.preferredSlotsForDoubles) {
    const prefs = occurrence.preferredSlotsForDoubles;
    if (prefs.preferred_days && 
        !prefs.preferred_days.includes(day)) {
      score += 10;
    }
    if (prefs.preferred_periods && 
        !prefs.preferred_periods.includes(periods[0].sequence)) {
      score += 10;
    }
  }
  
  return score;
}

/**
 * Assign consecutive slots (marks them as group)
 */
assignConsecutiveSlots(day, periods, occurrence) {
  const groupId = crypto.randomUUID();
  
  return periods.map((period, position) => ({
    classId: occurrence.classId,
    sectionId: occurrence.sectionId,
    subjectId: occurrence.subjectId,
    teacherId: occurrence.teacherId,
    day,
    periodId: period.id,
    roomId: occurrence.roomId || null,
    isConsecutiveStart: position === 0,
    consecutiveGroupId: groupId,
    consecutivePosition: position + 1,
    totalInGroup: periods.length
  }));
}
```

### Phase 4: Validation

```javascript
// NEW FILE: consecutive-validator.js

class ConsecutiveValidator {
  
  /**
   * Post-generation validation
   * Ensures all double/triple periods are actually consecutive
   */
  static validate(schedule, subjectConfigs) {
    const errors = [];
    const warnings = [];
    
    // Group by class + subject
    const grouped = {};
    for (const slot of schedule) {
      const key = `${slot.classId}-${slot.subjectId}`;
      if (!grouped[key]) grouped[key] = [];
      grouped[key].push(slot);
    }
    
    for (const [key, slots] of Object.entries(grouped)) {
      const [classId, subjectId] = key.split('-');
      const config = subjectConfigs.find(
        c => c.class_id === classId && c.subject_id === subjectId
      );
      
      if (!config) continue;
      
      const pattern = config.lesson_pattern || 
        LessonPatternParser.inferPattern(config);
      const parsed = LessonPatternParser.parsePattern(pattern);
      
      if (!parsed) continue;
      
      // Find actual consecutive groups in schedule
      const dayGroups = {};
      for (const slot of slots) {
        if (!dayGroups[slot.day]) dayGroups[slot.day] = [];
        dayGroups[slot.day].push(slot);
      }
      
      let doublesFound = 0;
      let triplesFound = 0;
      
      for (const [day, daySlots] of Object.entries(dayGroups)) {
        daySlots.sort(
          (a, b) => a.periodSequence - b.periodSequence
        );
        
        // Count consecutive groups
        let consecutiveCount = 1;
        for (let i = 1; i < daySlots.length; i++) {
          if (daySlots[i].periodSequence === 
              daySlots[i-1].periodSequence + 1) {
            consecutiveCount++;
          } else {
            if (consecutiveCount === 2) doublesFound++;
            if (consecutiveCount >= 3) triplesFound++;
            consecutiveCount = 1;
          }
        }
        if (consecutiveCount === 2) doublesFound++;
        if (consecutiveCount >= 3) triplesFound++;
      }
      
      // Validate against expected pattern
      if (doublesFound < parsed.doubleCount) {
        errors.push({
          type: 'MISSING_DOUBLE_PERIOD',
          classId, subjectId, pattern,
          expected: parsed.doubleCount,
          found: doublesFound,
          message: `Expected ${parsed.doubleCount} double periods ` +
            `but found ${doublesFound}`
        });
      }
      
      if (triplesFound < parsed.tripleCount) {
        errors.push({
          type: 'MISSING_TRIPLE_PERIOD',
          classId, subjectId, pattern,
          expected: parsed.tripleCount,
          found: triplesFound,
          message: `Expected ${parsed.tripleCount} triple periods ` +
            `but found ${triplesFound}`
        });
      }
    }
    
    return { valid: errors.length === 0, errors, warnings };
  }
}

module.exports = ConsecutiveValidator;
```

### Phase 5: API Endpoints

```javascript
// ADD TO: timetable-generator.routes.js

// Lesson Pattern CRUD
router.get('/lesson-patterns', controller.getLessonPatterns);
router.post('/lesson-patterns', controller.createLessonPattern);
router.put('/lesson-patterns/:id', controller.updateLessonPattern);
router.delete('/lesson-patterns/:id', controller.deleteLessonPattern);

// Apply pattern to subject
router.post(
  '/subject-configurations/:configId/apply-pattern',
  controller.applyPatternToSubject
);

// Validate patterns before generation
router.post(
  '/validate-patterns',
  controller.validatePatterns
);
```

---

# 📊 QUESTION 2: Class Divisions, Groups & Electives

## Current State Analysis

```
CURRENT STATUS: 0% Complete ❌ CRITICAL GAP

What EXISTS:
✅ class_id + section_id (basic: 10A, 10B, 10C)
✅ class_subject_teacher mapping

What's COMPLETELY MISSING:
❌ Sub-groups within sections (Hindi Group / Sanskrit Group)
❌ Joined lessons across sections (Assembly, PT combined)
❌ Co-teaching support (Main Teacher + Lab Assistant)
❌ Stream management (Science/Commerce/Arts for 11-12)
❌ Elective subject choices (Computer Science OR Biology)
❌ Student-group tracking
❌ Mutually exclusive subject scheduling
```

## Why This Is CRITICAL for Indian Schools

```
CBSE REQUIREMENT - Class 10:
  Section 10A (40 students):
  ├── ALL students: English, Math, Science, Social
  ├── GROUP 1 (25 students): Hindi as 2nd language
  └── GROUP 2 (15 students): Sanskrit as 2nd language
  
  Hindi Group and Sanskrit Group MUST be scheduled at 
  SAME TIME so students can attend their chosen language

CBSE REQUIREMENT - Class 11:
  Stream: Science
  ├── Core: Physics, Chemistry, English, PE
  ├── Elective 1: Mathematics OR Biology (student chooses)
  └── Elective 2: Computer Science OR Informatics (student chooses)
  
  Math and Biology sections run at SAME TIME
  CompSci and IP sections run at SAME TIME

COMBINED LESSONS:
  Assembly: ALL sections of ALL classes (1-12) together
  PT/Sports: 10A Boys + 10B Boys combined
  Music: 9A + 9B combined (resource teacher shared)
```

## Implementation Plan

### Phase 1: Database Schema

```sql
-- MIGRATION: 006_class_divisions_groups.sql

-- ============================================
-- 1. STUDENT GROUPS (Sub-divisions within sections)
-- ============================================
CREATE TABLE student_groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  class_id UUID NOT NULL REFERENCES classes(id),
  section_id UUID REFERENCES sections(id),
  group_name VARCHAR(100) NOT NULL,
  -- e.g., "Hindi Group", "Sanskrit Group", "Lab Group A"
  group_type VARCHAR(50) NOT NULL DEFAULT 'language',
  -- Types: 'language', 'elective', 'lab', 'gender', 
  --        'ability', 'custom'
  subject_id UUID REFERENCES subjects(id),
  -- Which subject this group is for (NULL if general group)
  max_students INTEGER,
  current_student_count INTEGER DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Student-Group Membership
CREATE TABLE student_group_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_group_id UUID NOT NULL REFERENCES student_groups(id) 
    ON DELETE CASCADE,
  student_id UUID NOT NULL REFERENCES students(id),
  enrolled_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(student_group_id, student_id)
);

-- ============================================
-- 2. MUTUALLY EXCLUSIVE GROUPS
--    (Groups that MUST be scheduled simultaneously)
-- ============================================
CREATE TABLE concurrent_group_sets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  class_id UUID NOT NULL REFERENCES classes(id),
  section_id UUID REFERENCES sections(id),
  set_name VARCHAR(100) NOT NULL,
  -- e.g., "Class 10A Language Elective"
  set_type VARCHAR(50) NOT NULL DEFAULT 'elective',
  -- Types: 'elective', 'language', 'stream', 'practical'
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE concurrent_group_set_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  set_id UUID NOT NULL REFERENCES concurrent_group_sets(id) 
    ON DELETE CASCADE,
  student_group_id UUID NOT NULL REFERENCES student_groups(id),
  subject_id UUID NOT NULL REFERENCES subjects(id),
  teacher_id UUID NOT NULL REFERENCES teachers(id),
  room_id UUID REFERENCES rooms(id),
  UNIQUE(set_id, student_group_id)
);

-- ============================================
-- 3. JOINED/COMBINED LESSONS
--    (Multiple sections/classes together)
-- ============================================
CREATE TABLE joined_lessons (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  lesson_name VARCHAR(200) NOT NULL,
  -- e.g., "Assembly", "Combined PT Boys", "Music 9A+9B"
  subject_id UUID NOT NULL REFERENCES subjects(id),
  teacher_id UUID REFERENCES teachers(id),
  -- NULL for assembly (no specific teacher)
  room_id UUID REFERENCES rooms(id),
  -- NULL for playground/assembly hall
  periods_per_week INTEGER NOT NULL DEFAULT 1,
  lesson_pattern VARCHAR(50) DEFAULT '1',
  -- Can be '2' for double period combined PT
  join_type VARCHAR(50) NOT NULL DEFAULT 'full_class',
  -- Types: 'full_class', 'boys_only', 'girls_only', 
  --        'group_based', 'grade_assembly'
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE joined_lesson_participants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  joined_lesson_id UUID NOT NULL REFERENCES joined_lessons(id) 
    ON DELETE CASCADE,
  class_id UUID NOT NULL REFERENCES classes(id),
  section_id UUID REFERENCES sections(id),
  -- NULL means ALL sections of this class
  student_group_id UUID REFERENCES student_groups(id),
  -- NULL means full section participates
  UNIQUE(joined_lesson_id, class_id, section_id, student_group_id)
);

-- ============================================
-- 4. CO-TEACHING ASSIGNMENTS
-- ============================================
CREATE TABLE co_teaching_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  class_id UUID NOT NULL REFERENCES classes(id),
  section_id UUID REFERENCES sections(id),
  subject_id UUID NOT NULL REFERENCES subjects(id),
  primary_teacher_id UUID NOT NULL REFERENCES teachers(id),
  co_teacher_ids UUID[] NOT NULL,
  -- Array of additional teacher UUIDs
  teaching_roles JSONB NOT NULL,
  -- e.g., {"primary": "Theory Teacher", 
  --        "co_teachers": [
  --          {"id": "uuid", "role": "Lab Assistant"},
  --          {"id": "uuid", "role": "Special Educator"}
  --        ]}
  applies_to_pattern VARCHAR(50) DEFAULT 'all',
  -- 'all' = both teachers every period
  -- 'double_only' = co-teacher only during double periods
  -- 'lab_only' = co-teacher only during lab periods
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- ============================================
-- 5. STREAMS (for Classes 11-12)
-- ============================================
CREATE TABLE academic_streams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  stream_name VARCHAR(100) NOT NULL,
  -- e.g., "Science", "Commerce", "Arts", "Humanities"
  applicable_classes UUID[] NOT NULL,
  -- Class IDs where this stream applies
  description TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stream_subjects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_id UUID NOT NULL REFERENCES academic_streams(id) 
    ON DELETE CASCADE,
  subject_id UUID NOT NULL REFERENCES subjects(id),
  subject_type VARCHAR(30) NOT NULL DEFAULT 'core',
  -- Types: 'core' (mandatory), 'elective' (choose), 
  --        'additional' (optional)
  elective_group INTEGER,
  -- Subjects with same elective_group number are 
  -- mutually exclusive (student picks one)
  -- e.g., Math=group1, Bio=group1 → student picks one
  periods_per_week INTEGER NOT NULL,
  is_active BOOLEAN DEFAULT true,
  UNIQUE(stream_id, subject_id)
);

-- ============================================
-- 6. STUDENT SUBJECT CHOICES
-- ============================================
CREATE TABLE student_subject_choices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id UUID NOT NULL REFERENCES students(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  stream_id UUID REFERENCES academic_streams(id),
  chosen_electives UUID[] NOT NULL,
  -- Array of subject IDs student has chosen
  approved_by UUID REFERENCES users(id),
  approved_at TIMESTAMP,
  status VARCHAR(30) DEFAULT 'pending',
  -- 'pending', 'approved', 'rejected', 'modified'
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(student_id, academic_year_id)
);

-- ============================================
-- 7. UPDATE class_subject_teacher for groups
-- ============================================
ALTER TABLE class_subject_teacher
ADD COLUMN student_group_id UUID REFERENCES student_groups(id),
ADD COLUMN is_joined_lesson BOOLEAN DEFAULT false,
ADD COLUMN joined_lesson_id UUID REFERENCES joined_lessons(id),
ADD COLUMN co_teaching_id UUID 
  REFERENCES co_teaching_assignments(id);

-- ============================================
-- INDEXES
-- ============================================
CREATE INDEX idx_student_groups_class 
  ON student_groups(class_id, section_id);
CREATE INDEX idx_student_groups_school_year 
  ON student_groups(school_id, academic_year_id);
CREATE INDEX idx_concurrent_groups_class 
  ON concurrent_group_sets(class_id, section_id);
CREATE INDEX idx_joined_lessons_school 
  ON joined_lessons(school_id, academic_year_id);
CREATE INDEX idx_stream_subjects_stream 
  ON stream_subjects(stream_id);
CREATE INDEX idx_student_choices_student 
  ON student_subject_choices(student_id, academic_year_id);
```

### Phase 2: Enhanced Workload Builder

```javascript
// MODIFIED: workload.builder.js

class EnhancedWorkloadBuilder {
  
  /**
   * Build workloads with group/division awareness
   */
  buildClassWorkloads(data) {
    const workloads = [];
    
    for (const classSection of data.classSections) {
      const workload = {
        classId: classSection.class_id,
        sectionId: classSection.section_id,
        subjects: [],
        groupedSubjects: [],    // NEW: Concurrent groups
        joinedLessons: [],      // NEW: Combined lessons
        coTeachingSubjects: []  // NEW: Co-teaching
      };
      
      // 1. Regular subjects (full section)
      const regularAssignments = data.assignments.filter(
        a => a.class_id === classSection.class_id &&
             a.section_id === classSection.section_id &&
             !a.student_group_id &&
             !a.is_joined_lesson
      );
      
      for (const assignment of regularAssignments) {
        workload.subjects.push(
          this.buildSubjectWorkload(assignment, data)
        );
      }
      
      // 2. Grouped subjects (mutually exclusive electives)
      const concurrentSets = data.concurrentGroupSets.filter(
        s => s.class_id === classSection.class_id &&
             s.section_id === classSection.section_id
      );
      
      for (const set of concurrentSets) {
        const groupMembers = data.concurrentGroupSetMembers
          .filter(m => m.set_id === set.id);
        
        workload.groupedSubjects.push({
          setId: set.id,
          setName: set.set_name,
          setType: set.set_type,
          // ALL these subjects must be at SAME time
          concurrentSubjects: groupMembers.map(m => ({
            subjectId: m.subject_id,
            teacherId: m.teacher_id,
            roomId: m.room_id,
            groupId: m.student_group_id,
            weeklyFrequency: this.getFrequency(
              m.subject_id, classSection, data
            )
          })),
          // They share the same time slots
          weeklyFrequency: this.getFrequency(
            groupMembers[0].subject_id, classSection, data
          )
        });
      }
      
      // 3. Joined lessons (combined sections)
      const joinedLessons = data.joinedLessons.filter(
        jl => data.joinedLessonParticipants.some(
          p => p.joined_lesson_id === jl.id &&
               p.class_id === classSection.class_id &&
               (p.section_id === classSection.section_id || 
                p.section_id === null)
        )
      );
      
      for (const jl of joinedLessons) {
        workload.joinedLessons.push({
          joinedLessonId: jl.id,
          lessonName: jl.lesson_name,
          subjectId: jl.subject_id,
          teacherId: jl.teacher_id,
          roomId: jl.room_id,
          weeklyFrequency: jl.periods_per_week,
          lessonPattern: jl.lesson_pattern,
          participants: data.joinedLessonParticipants
            .filter(p => p.joined_lesson_id === jl.id)
            .map(p => ({
              classId: p.class_id,
              sectionId: p.section_id,
              groupId: p.student_group_id
            }))
        });
      }
      
      // 4. Co-teaching subjects
      const coTeaching = data.coTeachingAssignments.filter(
        ct => ct.class_id === classSection.class_id &&
              ct.section_id === classSection.section_id
      );
      
      for (const ct of coTeaching) {
        workload.coTeachingSubjects.push({
          subjectId: ct.subject_id,
          primaryTeacherId: ct.primary_teacher_id,
          coTeacherIds: ct.co_teacher_ids,
          teachingRoles: ct.teaching_roles,
          appliesToPattern: ct.applies_to_pattern
        });
      }
      
      workloads.push(workload);
    }
    
    return workloads;
  }
}
```

### Phase 3: Concurrent Group Scheduler

```javascript
// NEW FILE: concurrent-group.scheduler.js

class ConcurrentGroupScheduler {
  
  /**
   * Schedule mutually exclusive groups
   * All subjects in a concurrent set get SAME day + period
   */
  scheduleGroupedSubjects(groupedSubject, existingSlots, 
    daysPerWeek, gradePeriods) {
    
    const results = [];
    const frequency = groupedSubject.weeklyFrequency;
    
    for (let occ = 0; occ < frequency; occ++) {
      const candidates = [];
      
      for (let day = 1; day <= daysPerWeek; day++) {
        for (const period of gradePeriods) {
          // Check if ALL concurrent subjects can fit here
          let allFit = true;
          
          for (const sub of groupedSubject.concurrentSubjects) {
            // Check teacher availability for each subject
            if (!this.isTeacherAvailable(
              sub.teacherId, day, period.id, existingSlots
            )) {
              allFit = false;
              break;
            }
            
            // Check room availability for each subject
            if (sub.roomId && !this.isRoomAvailable(
              sub.roomId, day, period.id, existingSlots
            )) {
              allFit = false;
              break;
            }
          }
          
          // Check class-section not already busy
          if (!this.isClassFree(
            groupedSubject.classId, 
            groupedSubject.sectionId,
            day, period.id, existingSlots
          )) {
            allFit = false;
          }
          
          if (allFit) {
            candidates.push({ day, period });
          }
        }
      }
      
      if (candidates.length === 0) {
        throw new Error(
          `Cannot schedule concurrent group ` +
          `"${groupedSubject.setName}" - ` +
          `no slot where ALL teachers are free simultaneously`
        );
      }
      
      // Pick best candidate
      const best = this.scoreCandidates(
        candidates, existingSlots, groupedSubject
      )[0];
      
      // Create slots for ALL concurrent subjects at SAME time
      for (const sub of groupedSubject.concurrentSubjects) {
        results.push({
          classId: groupedSubject.classId,
          sectionId: groupedSubject.sectionId,
          subjectId: sub.subjectId,
          teacherId: sub.teacherId,
          roomId: sub.roomId,
          groupId: sub.groupId,
          day: best.day,
          periodId: best.period.id,
          isConcurrentGroup: true,
          concurrentSetId: groupedSubject.setId
        });
      }
    }
    
    return results;
  }
}

module.exports = ConcurrentGroupScheduler;
```

### Phase 4: Joined Lesson Scheduler

```javascript
// NEW FILE: joined-lesson.scheduler.js

class JoinedLessonScheduler {
  
  /**
   * Schedule combined lessons (Assembly, Combined PT, etc.)
   * Must find slots where ALL participating classes are free
   */
  scheduleJoinedLesson(joinedLesson, existingSlots, 
    allClassSchedules, daysPerWeek, periodsByStructure) {
    
    const results = [];
    const participants = joinedLesson.participants;
    
    // Find common periods across all participant classes
    // (they may have different structures!)
    const commonPeriods = this.findCommonPeriods(
      participants, periodsByStructure
    );
    
    if (commonPeriods.length === 0) {
      throw new Error(
        `Joined lesson "${joinedLesson.lessonName}": ` +
        `No common periods across participating classes`
      );
    }
    
    const frequency = joinedLesson.weeklyFrequency;
    
    for (let occ = 0; occ < frequency; occ++) {
      const candidates = [];
      
      for (let day = 1; day <= daysPerWeek; day++) {
        for (const period of commonPeriods) {
          let allFree = true;
          
          // Check EVERY participating class is free
          for (const participant of participants) {
            const classBusy = existingSlots.some(
              s => s.classId === participant.classId &&
                   (participant.sectionId === null || 
                    s.sectionId === participant.sectionId) &&
                   s.day === day &&
                   s.periodId === period.id
            );
            
            if (classBusy) {
              allFree = false;
              break;
            }
          }
          
          // Check teacher is free (if specified)
          if (joinedLesson.teacherId) {
            const teacherBusy = existingSlots.some(
              s => s.teacherId === joinedLesson.teacherId &&
                   s.day === day &&
                   s.periodId === period.id
            );
            if (teacherBusy) allFree = false;
          }
          
          // Check room is free (if specified)
          if (joinedLesson.roomId) {
            const roomBusy = existingSlots.some(
              s => s.roomId === joinedLesson.roomId &&
                   s.day === day &&
                   s.periodId === period.id
            );
            if (roomBusy) allFree = false;
          }
          
          if (allFree) {
            candidates.push({ day, period });
          }
        }
      }
      
      if (candidates.length === 0) {
        throw new Error(
          `Joined lesson "${joinedLesson.lessonName}": ` +
          `Cannot find a slot where all ` +
          `${participants.length} classes are free`
        );
      }
      
      const best = candidates[0]; // Simplest scoring
      
      // Create slot for EACH participating class
      for (const participant of participants) {
        const sections = participant.sectionId 
          ? [participant.sectionId]
          : this.getAllSections(participant.classId);
        
        for (const sectionId of sections) {
          results.push({
            classId: participant.classId,
            sectionId,
            subjectId: joinedLesson.subjectId,
            teacherId: joinedLesson.teacherId,
            roomId: joinedLesson.roomId,
            groupId: participant.groupId,
            day: best.day,
            periodId: best.period.id,
            isJoinedLesson: true,
            joinedLessonId: joinedLesson.id
          });
        }
      }
    }
    
    return results;
  }
}

module.exports = JoinedLessonScheduler;
```

### Phase 5: API Endpoints

```javascript
// ADD TO routes

// Student Groups
router.get('/student-groups', controller.getStudentGroups);
router.post('/student-groups', controller.createStudentGroup);
router.put('/student-groups/:id', controller.updateStudentGroup);
router.delete('/student-groups/:id', controller.deleteStudentGroup);
router.post(
  '/student-groups/:id/members', 
  controller.addGroupMembers
);

// Concurrent Group Sets
router.get('/concurrent-sets', controller.getConcurrentSets);
router.post('/concurrent-sets', controller.createConcurrentSet);
router.put('/concurrent-sets/:id', controller.updateConcurrentSet);

// Joined Lessons
router.get('/joined-lessons', controller.getJoinedLessons);
router.post('/joined-lessons', controller.createJoinedLesson);
router.put('/joined-lessons/:id', controller.updateJoinedLesson);

// Streams
router.get('/streams', controller.getStreams);
router.post('/streams', controller.createStream);

// Student Choices
router.get(
  '/student-choices/:studentId', 
  controller.getStudentChoices
);
router.post('/student-choices', controller.saveStudentChoices);
router.post(
  '/student-choices/:id/approve', 
  controller.approveChoices
);

// Co-Teaching
router.get('/co-teaching', controller.getCoTeachingAssignments);
router.post('/co-teaching', controller.createCoTeaching);
```

---

# 📊 QUESTION 3: Constraint Engine

## Current State Analysis

```
CURRENT STATUS: 47% Overall ⚠️

HARD CONSTRAINTS: 11/11 ✅ (100%)
  ✅ Teacher availability
  ✅ Teacher conflict (no double-booking)
  ✅ Class-section conflict
  ✅ Room conflict
  ✅ Lab requirements
  ✅ Max periods per day (grade level)
  ✅ Max subject per day
  ✅ No consecutive same subject
  ✅ Heavy subject - first period rule (⚠️ flag only)
  ✅ Heavy subject - last period rule
  ✅ No consecutive heavy subjects

SOFT CONSTRAINTS: 3/13 ⚠️ (23%)
  ⚠️ Balanced day loads (basic)
  ⚠️ Avoid consecutive heavy (penalty only)
  ⚠️ Random distribution (shuffle only)
  ❌ 10+ missing soft constraints

PEDAGOGICAL: 2/10 ❌ (20%)
  ❌ No Math after PT
  ❌ Heavy subjects in morning
  ❌ Distribute subjects evenly across week
  ❌ No heavy after lunch/recess
  ❌ Easy subjects last period (flag exists, unused)
  ❌ Post-break assignment (flag exists, unused)
  ❌ Subject pair preferences
  ❌ Teacher preferences (morning/afternoon)
  ❌ Gap minimization for teachers
  ❌ Workload balancing across teachers
```

## Implementation Plan

### Phase 1: Constraint Registry System

```javascript
// NEW FILE: constraint-registry.js

/**
 * Central registry for ALL constraints
 * Each constraint has:
 * - type: 'hard' | 'soft'
 * - priority: 0 (highest) to 10 (lowest)
 * - weight: scoring weight for soft constraints
 * - enabled: can be toggled per school
 * - validator: function that checks the constraint
 */
class ConstraintRegistry {
  constructor() {
    this.constraints = new Map();
    this.registerDefaults();
  }
  
  registerDefaults() {
    // =========================================
    // HARD CONSTRAINTS (priority 0-2)
    // MUST NEVER be violated
    // =========================================
    
    this.register('TEACHER_CONFLICT', {
      type: 'hard',
      priority: 0,
      name: 'Teacher Double-Booking Prevention',
      description: 'Teacher cannot teach 2 classes at same time',
      enabled: true, // Cannot be disabled
      locked: true,  // Cannot be disabled by school
      validator: (slot, schedule, data) => {
        const conflict = schedule.find(
          s => s.teacherId === slot.teacherId &&
               s.day === slot.day &&
               s.periodId === slot.periodId &&
               s.id !== slot.id
        );
        return {
          valid: !conflict,
          message: conflict ? 
            `Teacher already teaching ${conflict.className}` +
            ` - ${conflict.subjectName}` : null,
          conflictingSlot: conflict
        };
      }
    });
    
    this.register('CLASS_CONFLICT', {
      type: 'hard',
      priority: 0,
      name: 'Class Double-Booking Prevention',
      description: 'Class cannot have 2 subjects at same time',
      enabled: true,
      locked: true,
      validator: (slot, schedule, data) => {
        const conflict = schedule.find(
          s => s.classId === slot.classId &&
               s.sectionId === slot.sectionId &&
               s.day === slot.day &&
               s.periodId === slot.periodId &&
               s.id !== slot.id
        );
        return {
          valid: !conflict,
          message: conflict ? 
            `Class already has ${conflict.subjectName}` : null
        };
      }
    });
    
    this.register('ROOM_CONFLICT', {
      type: 'hard',
      priority: 0,
      name: 'Room Double-Booking Prevention',
      description: 'Room cannot host 2 classes at same time',
      enabled: true,
      locked: true,
      validator: (slot, schedule, data) => {
        if (!slot.roomId) return { valid: true };
        const conflict = schedule.find(
          s => s.roomId === slot.roomId &&
               s.day === slot.day &&
               s.periodId === slot.periodId &&
               s.id !== slot.id
        );
        return {
          valid: !conflict,
          message: conflict ? 
            `Room occupied by ${conflict.className}` : null
        };
      }
    });
    
    this.register('TEACHER_AVAILABILITY', {
      type: 'hard',
      priority: 0,
      name: 'Teacher Availability',
      description: 'Teacher must be available at scheduled time',
      enabled: true,
      locked: true,
      validator: (slot, schedule, data) => {
        const available = data.availabilityMatrix
          .isAvailable(slot.teacherId, slot.day, slot.periodId);
        return {
          valid: available,
          message: !available ? 
            'Teacher not available at this time' : null
        };
      }
    });
    
    this.register('LAB_REQUIREMENT', {
      type: 'hard',
      priority: 1,
      name: 'Lab Room Requirement',
      description: 'Lab subjects must be in lab rooms',
      enabled: true,
      locked: true,
      validator: (slot, schedule, data) => {
        if (!slot.requiresLab) return { valid: true };
        const room = data.rooms.find(r => r.id === slot.roomId);
        const isLab = room && room.room_type === 'lab';
        return {
          valid: isLab,
          message: !isLab ? 
            'Lab subject must be assigned to lab room' : null
        };
      }
    });
    
    this.register('MAX_PERIODS_PER_DAY', {
      type: 'hard',
      priority: 1,
      name: 'Max Periods Per Day',
      description: 'Grade-level maximum periods per day',
      enabled: true,
      validator: (slot, schedule, data) => {
        const classPeriodsToday = schedule.filter(
          s => s.classId === slot.classId &&
               s.sectionId === slot.sectionId &&
               s.day === slot.day
        ).length;
        const max = data.gradeRules
          ?.find(r => r.class_id === slot.classId)
          ?.max_periods_per_day || 8;
        return {
          valid: classPeriodsToday < max,
          message: classPeriodsToday >= max ? 
            `Already ${classPeriodsToday}/${max} periods today` 
            : null
        };
      }
    });
    
    this.register('MAX_SUBJECT_PER_DAY', {
      type: 'hard',
      priority: 1,
      name: 'Max Same Subject Per Day',
      description: 'Limits same subject repetitions per day',
      enabled: true,
      validator: (slot, schedule, data) => {
        const subjectToday = schedule.filter(
          s => s.classId === slot.classId &&
               s.sectionId === slot.sectionId &&
               s.subjectId === slot.subjectId &&
               s.day === slot.day
        ).length;
        const config = data.subjectConfigs?.find(
          c => c.subject_id === slot.subjectId && 
               c.class_id === slot.classId
        );
        const max = config?.max_per_day || 2;
        return {
          valid: subjectToday < max,
          message: subjectToday >= max ? 
            `${slot.subjectName} already at max ${max}/day` 
            : null
        };
      }
    });
    
    this.register('ROOM_CAPACITY', {
      type: 'hard',
      priority: 1,
      name: 'Room Capacity Check',
      description: 'Room must have enough seats for class size',
      enabled: true,
      validator: (slot, schedule, data) => {
        if (!slot.roomId) return { valid: true };
        const room = data.rooms.find(r => r.id === slot.roomId);
        const sectionSize = data.sectionSizes
          ?.[`${slot.classId}-${slot.sectionId}`] || 0;
        const fits = !room?.capacity || 
          room.capacity >= sectionSize;
        return {
          valid: fits,
          message: !fits ? 
            `Room capacity ${room.capacity} < ` +
            `class size ${sectionSize}` : null
        };
      }
    });

    // =========================================
    // SOFT CONSTRAINTS (priority 3-7)
    // Should be satisfied, scored as penalties
    // =========================================
    
    this.register('NO_HEAVY_AFTER_PT', {
      type: 'soft',
      priority: 3,
      weight: 25,
      name: 'No Heavy Subject After PT/Sports',
      description: 'Math/Science should not follow PE',
      enabled: true,
      validator: (slot, schedule, data) => {
        if (!slot.isHeavy) return { valid: true, penalty: 0 };
        
        const prevPeriod = this.getPreviousPeriod(
          slot.periodId, data.periods
        );
        if (!prevPeriod) return { valid: true, penalty: 0 };
        
        const prevSlot = schedule.find(
          s => s.classId === slot.classId &&
               s.sectionId === slot.sectionId &&
               s.day === slot.day &&
               s.periodId === prevPeriod.id
        );
        
        const isPT = prevSlot && 
          (prevSlot.subjectName?.toLowerCase().includes('physical') ||
           prevSlot.subjectName?.toLowerCase().includes('sports') ||
           prevSlot.subjectName?.toLowerCase().includes('pt'));
        
        return {
          valid: !isPT,
          penalty: isPT ? 25 : 0,
          message: isPT ? 
            'Heavy subject placed after PT/Sports' : null
        };
      }
    });
    
    this.register('HEAVY_IN_MORNING', {
      type: 'soft',
      priority: 3,
      weight: 20,
      name: 'Heavy Subjects in Morning',
      description: 'Math/Science should be in periods 1-4',
      enabled: true,
      validator: (slot, schedule, data) => {
        if (!slot.isHeavy) return { valid: true, penalty: 0 };
        
        const period = data.periods.find(
          p => p.id === slot.periodId
        );
        const periodIdx = period?.sequence || 0;
        
        // Ideal: periods 1-4 (morning)
        // Acceptable: periods 5-6
        // Penalize: periods 7-8
        let penalty = 0;
        if (periodIdx >= 7) penalty = 20;
        else if (periodIdx >= 5) penalty = 10;
        
        return {
          valid: periodIdx <= 4,
          penalty,
          message: periodIdx > 4 ? 
            `Heavy subject in period ${periodIdx} ` +
            `(prefer morning)` : null
        };
      }
    });
    
    this.register('NO_HEAVY_AFTER_LUNCH', {
      type: 'soft',
      priority: 3,
      weight: 20,
      name: 'No Heavy Subject After Lunch/Break',
      description: 'Avoid Math/Science right after lunch break',
      enabled: true,
      validator: (slot, schedule, data) => {
        if (!slot.isHeavy) return { valid: true, penalty: 0 };
        
        const prevPeriod = this.getPreviousPeriodInAll(
          slot.periodId, data.allPeriods
        );
        const isAfterBreak = prevPeriod && prevPeriod.is_break;
        
        return {
          valid: !isAfterBreak,
          penalty: isAfterBreak ? 20 : 0,
          message: isAfterBreak ? 
            'Heavy subject placed right after break' : null
        };
      }
    });
    
    this.register('EVEN_DISTRIBUTION', {
      type: 'soft',
      priority: 4,
      weight: 15,
      name: 'Even Subject Distribution',
      description: 'Spread subjects evenly across week',
      enabled: true,
      validator: (slot, schedule, data) => {
        // Check if this subject already scheduled on adjacent days
        const subjectDays = schedule
          .filter(
            s => s.classId === slot.classId &&
                 s.sectionId === slot.sectionId &&
                 s.subjectId === slot.subjectId
          )
          .map(s => s.day);
        
        let penalty = 0;
        
        // Penalty for scheduling on consecutive days
        if (subjectDays.includes(slot.day - 1)) penalty += 15;
        if (subjectDays.includes(slot.day + 1)) penalty += 15;
        
        // Penalty for clustering (e.g., Mon-Tue-Wed)
        if (subjectDays.includes(slot.day - 1) && 
            subjectDays.includes(slot.day - 2)) {
          penalty += 25;
        }
        
        return {
          valid: penalty === 0,
          penalty,
          message: penalty > 0 ? 
            'Subject clustered on consecutive days' : null
        };
      }
    });
    
    this.register('EASY_LAST_PERIOD', {
      type: 'soft',
      priority: 5,
      weight: 10,
      name: 'Easy Subjects Last Period',
      description: 'Art, Music, Library should be at end of day',
      enabled: true,
      validator: (slot, schedule, data) => {
        const period = data.periods.find(
          p => p.id === slot.periodId
        );
        const isLastPeriod = period?.sequence === 
          data.teachingPeriods.length;
        
        // If last period, prefer easy subjects
        if (isLastPeriod && slot.isHeavy) {
          return {
            valid: false,
            penalty: 10,
            message: 'Heavy subject in last period'
          };
        }
        
        // If easy subject, prefer later periods
        if (slot.isEasy && period?.sequence <= 3) {
          return {
            valid: false,
            penalty: 5,
            message: 'Easy subject in early morning slot'
          };
        }
        
        return { valid: true, penalty: 0 };
      }
    });
    
    this.register('TEACHER_GAP_MINIMIZATION', {
      type: 'soft',
      priority: 5,
      weight: 12,
      name: 'Teacher Gap Minimization',
      description: 'Minimize free periods between classes',
      enabled: true,
      validator: (slot, schedule, data) => {
        const teacherSlots = schedule
          .filter(
            s => s.teacherId === slot.teacherId && 
                 s.day === slot.day
          )
          .sort(
            (a, b) => a.periodSequence - b.periodSequence
          );
        
        if (teacherSlots.length === 0) {
          return { valid: true, penalty: 0 };
        }
        
        // Calculate gap if this slot is added
        const allSlots = [...teacherSlots, slot]
          .sort(
            (a, b) => a.periodSequence - b.periodSequence
          );
        
        let totalGaps = 0;
        for (let i = 1; i < allSlots.length; i++) {
          const gap = allSlots[i].periodSequence - 
                      allSlots[i-1].periodSequence - 1;
          totalGaps += Math.max(0, gap);
        }
        
        return {
          valid: totalGaps === 0,
          penalty: totalGaps * 12,
          message: totalGaps > 0 ? 
            `Creates ${totalGaps} gap period(s) for teacher` 
            : null
        };
      }
    });
    
    this.register('NO_CONSECUTIVE_SAME_SUBJECT', {
      type: 'soft', // Can be hard per school config
      priority: 4,
      weight: 30,
      name: 'No Consecutive Same Subject',
      description: 'Avoid same subject in back-to-back periods',
      enabled: true,
      configurable: true, // School can make this hard/soft
      validator: (slot, schedule, data) => {
        const prevPeriod = this.getPreviousPeriod(
          slot.periodId, data.periods
        );
        const nextPeriod = this.getNextPeriod(
          slot.periodId, data.periods
        );
        
        let violation = false;
        
        // Check previous period
        if (prevPeriod) {
          const prevSlot = schedule.find(
            s => s.classId === slot.classId &&
                 s.sectionId === slot.sectionId &&
                 s.day === slot.day &&
                 s.periodId === prevPeriod.id
          );
          if (prevSlot && 
              prevSlot.subjectId === slot.subjectId &&
              !slot.isConsecutiveStart) {
            // Exception: If this IS a double period, 
            // consecutive is intended
            violation = true;
          }
        }
        
        return {
          valid: !violation,
          penalty: violation ? 30 : 0,
          message: violation ? 
            'Same subject in consecutive periods' : null
        };
      }
    });
    
    this.register('TEACHER_MAX_CONSECUTIVE', {
      type: 'soft',
      priority: 4,
      weight: 15,
      name: 'Teacher Max Consecutive Periods',
      description: 'Teacher should not teach 4+ periods in a row',
      enabled: true,
      validator: (slot, schedule, data) => {
        const teacherSlots = schedule
          .filter(
            s => s.teacherId === slot.teacherId && 
                 s.day === slot.day
          )
          .sort(
            (a, b) => a.periodSequence - b.periodSequence
          );
        
        // Count current consecutive streak
        let consecutive = 1;
        const thisSeq = slot.periodSequence;
        
        // Count backwards
        let seq = thisSeq - 1;
        while (teacherSlots.find(
          s => s.periodSequence === seq
        )) {
          consecutive++;
          seq--;
        }
        
        // Count forwards
        seq = thisSeq + 1;
        while (teacherSlots.find(
          s => s.periodSequence === seq
        )) {
          consecutive++;
          seq++;
        }
        
        const maxConsecutive = data.schoolConfig
          ?.max_teacher_consecutive || 3;
        
        return {
          valid: consecutive <= maxConsecutive,
          penalty: consecutive > maxConsecutive ? 
            (consecutive - maxConsecutive) * 15 : 0,
          message: consecutive > maxConsecutive ? 
            `Teacher has ${consecutive} consecutive periods ` +
            `(max ${maxConsecutive})` : null
        };
      }
    });
    
    this.register('TEACHER_LUNCH_GUARANTEE', {
      type: 'soft',
      priority: 3,
      weight: 30,
      name: 'Teacher Lunch Break Guarantee',
      description: 'Teacher must have at least 1 break period',
      enabled: true,
      validator: (slot, schedule, data) => {
        // Check AFTER scheduling: does teacher have any 
        // break period free?
        const lunchPeriods = data.allPeriods.filter(
          p => p.is_break
        );
        
        // This is a post-schedule validation
        // In isSlotValid, we check: if assigning this slot 
        // means teacher teaches through ALL breaks
        const teacherSlots = schedule.filter(
          s => s.teacherId === slot.teacherId && 
               s.day === slot.day
        );
        
        // Get periods adjacent to breaks
        const periodsAroundBreaks = [];
        for (const bp of lunchPeriods) {
          const before = data.periods.find(
            p => p.sequence === bp.sequence - 1
          );
          const after = data.periods.find(
            p => p.sequence === bp.sequence + 1
          );
          if (before) periodsAroundBreaks.push(before.id);
          if (after) periodsAroundBreaks.push(after.id);
        }
        
        // If teacher teaches all periods around break, 
        // they might miss lunch
        const teachesAllAroundBreak = periodsAroundBreaks.every(
          pid => teacherSlots.some(s => s.periodId === pid) || 
                 slot.periodId === pid
        );
        
        return {
          valid: !teachesAllAroundBreak,
          penalty: teachesAllAroundBreak ? 30 : 0,
          message: teachesAllAroundBreak ? 
            'Teacher may miss lunch break' : null
        };
      }
    });
    
    this.register('CLASS_TEACHER_FIRST_PERIOD', {
      type: 'soft',
      priority: 6,
      weight: 8,
      name: 'Class Teacher First Period',
      description: 'Class teacher should have first period ' +
        'with own class at least once a week',
      enabled: true,
      validator: (slot, schedule, data) => {
        // Only check if slot is period 1
        const period = data.periods.find(
          p => p.id === slot.periodId
        );
        if (period?.sequence !== 1) {
          return { valid: true, penalty: 0 };
        }
        
        // Check if slot's teacher is the class teacher
        const classTeacher = data.classTeachers?.find(
          ct => ct.class_id === slot.classId && 
                ct.section_id === slot.sectionId
        );
        
        if (!classTeacher) return { valid: true, penalty: 0 };
        
        // Prefer class teacher for first period
        if (slot.teacherId !== classTeacher.teacher_id) {
          return {
            valid: false,
            penalty: 8,
            message: 'First period not assigned to class teacher'
          };
        }
        
        return { valid: true, penalty: 0 };
      }
    });

    this.register('BUILDING_TRANSFER', {
      type: 'hard',
      priority: 2,
      name: 'Building Transfer Check',
      description: 'No consecutive periods in different buildings',
      enabled: true,
      validator: (slot, schedule, data) => {
        if (!slot.roomId) return { valid: true };
        
        const prevPeriod = this.getPreviousPeriod(
          slot.periodId, data.periods
        );
        if (!prevPeriod) return { valid: true };
        
        const prevSlot = schedule.find(
          s => s.classId === slot.classId &&
               s.sectionId === slot.sectionId &&
               s.day === slot.day &&
               s.periodId === prevPeriod.id
        );
        
        if (!prevSlot || !prevSlot.roomId) {
          return { valid: true };
        }
        
        const currentRoom = data.rooms.find(
          r => r.id === slot.roomId
        );
        const prevRoom = data.rooms.find(
          r => r.id === prevSlot.roomId
        );
        
        if (currentRoom?.building && prevRoom?.building &&
            currentRoom.building !== prevRoom.building) {
          return {
            valid: false,
            message: `Building transfer: ` +
              `${prevRoom.building} → ${currentRoom.building} ` +
              `in consecutive periods`
          };
        }
        
        return { valid: true };
      }
    });
  }
  
  /**
   * Register a new constraint
   */
  register(id, constraint) {
    this.constraints.set(id, { id, ...constraint });
  }
  
  /**
   * Get all hard constraints
   */
  getHardConstraints() {
    return [...this.constraints.values()]
      .filter(c => c.type === 'hard' && c.enabled)
      .sort((a, b) => a.priority - b.priority);
  }
  
  /**
   * Get all soft constraints
   */
  getSoftConstraints() {
    return [...this.constraints.values()]
      .filter(c => c.type === 'soft' && c.enabled)
      .sort((a, b) => a.priority - b.priority);
  }
  
  /**
   * Validate a slot against ALL hard constraints
   * Returns false if ANY hard constraint is violated
   */
  validateHard(slot, schedule, data) {
    for (const constraint of this.getHardConstraints()) {
      const result = constraint.validator(slot, schedule, data);
      if (!result.valid) {
        return {
          valid: false,
          violatedConstraint: constraint.id,
          constraintName: constraint.name,
          message: result.message,
          conflictingSlot: result.conflictingSlot
        };
      }
    }
    return { valid: true };
  }
  
  /**
   * Score a slot against ALL soft constraints
   * Returns total penalty (lower = better)
   */
  scoreSoft(slot, schedule, data) {
    let totalPenalty = 0;
    const violations = [];
    
    for (const constraint of this.getSoftConstraints()) {
      const result = constraint.validator(slot, schedule, data);
      if (!result.valid || result.penalty > 0) {
        totalPenalty += result.penalty || constraint.weight;
        violations.push({
          constraintId: constraint.id,
          constraintName: constraint.name,
          penalty: result.penalty || constraint.weight,
          message: result.message
        });
      }
    }
    
    return { totalPenalty, violations };
  }
  
  // Helper methods
  getPreviousPeriod(periodId, periods) {
    const current = periods.find(p => p.id === periodId);
    if (!current) return null;
    return periods
      .filter(p => p.sequence < current.sequence && p.is_teaching)
      .sort((a, b) => b.sequence - a.sequence)[0] || null;
  }
  
  getNextPeriod(periodId, periods) {
    const current = periods.find(p => p.id === periodId);
    if (!current) return null;
    return periods
      .filter(p => p.sequence > current.sequence && p.is_teaching)
      .sort((a, b) => a.sequence - b.sequence)[0] || null;
  }
  
  getPreviousPeriodInAll(periodId, allPeriods) {
    const current = allPeriods.find(p => p.id === periodId);
    if (!current) return null;
    return allPeriods
      .filter(p => p.sequence === current.sequence - 1)[0] || null;
  }
}

module.exports = ConstraintRegistry;
```

### Phase 2: School-Level Constraint Configuration

```sql
-- MIGRATION: 007_constraint_configuration.sql

CREATE TABLE school_constraint_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  constraint_id VARCHAR(100) NOT NULL,
  -- e.g., 'NO_HEAVY_AFTER_PT', 'EVEN_DISTRIBUTION'
  constraint_type VARCHAR(10) NOT NULL DEFAULT 'soft',
  -- School can upgrade soft → hard or downgrade
  is_enabled BOOLEAN DEFAULT true,
  weight INTEGER DEFAULT 10,
  -- Custom weight override
  custom_config JSONB DEFAULT NULL,
  -- Constraint-specific configuration
  -- e.g., {"max_consecutive": 3, "subjects": ["math","science"]}
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(school_id, academic_year_id, constraint_id)
);

-- Insert defaults for new school
INSERT INTO school_constraint_configs 
  (school_id, academic_year_id, constraint_id, 
   constraint_type, is_enabled, weight) 
VALUES
  ('school', 'year', 'TEACHER_CONFLICT', 'hard', true, 100),
  ('school', 'year', 'CLASS_CONFLICT', 'hard', true, 100),
  ('school', 'year', 'ROOM_CONFLICT', 'hard', true, 100),
  ('school', 'year', 'TEACHER_AVAILABILITY', 'hard', true, 100),
  ('school', 'year', 'LAB_REQUIREMENT', 'hard', true, 90),
  ('school', 'year', 'MAX_PERIODS_PER_DAY', 'hard', true, 90),
  ('school', 'year', 'ROOM_CAPACITY', 'hard', true, 80),
  ('school', 'year', 'BUILDING_TRANSFER', 'hard', true, 80),
  ('school', 'year', 'NO_HEAVY_AFTER_PT', 'soft', true, 25),
  ('school', 'year', 'HEAVY_IN_MORNING', 'soft', true, 20),
  ('school', 'year', 'NO_HEAVY_AFTER_LUNCH', 'soft', true, 20),
  ('school', 'year', 'EVEN_DISTRIBUTION', 'soft', true, 15),
  ('school', 'year', 'EASY_LAST_PERIOD', 'soft', true, 10),
  ('school', 'year', 'TEACHER_GAP_MINIMIZATION', 'soft', true, 12),
  ('school', 'year', 'TEACHER_MAX_CONSECUTIVE', 'soft', true, 15),
  ('school', 'year', 'TEACHER_LUNCH_GUARANTEE', 'soft', true, 30),
  ('school', 'year', 'CLASS_TEACHER_FIRST_PERIOD', 'soft', 
    true, 8);
```

---

# 📊 QUESTIONS 4-5: Grade-Specific Structures & Rooms

## Current State: Q4 = 100% ✅, Q5 = 40% ⚠️

### Room System Implementation

```sql
-- MIGRATION: 008_enhanced_rooms.sql

-- Fix 1: Add missing columns to rooms
ALTER TABLE rooms
ADD COLUMN IF NOT EXISTS building VARCHAR(100),
ADD COLUMN IF NOT EXISTS floor INTEGER,
ADD COLUMN IF NOT EXISTS wing VARCHAR(50),
ADD COLUMN IF NOT EXISTS is_accessible BOOLEAN DEFAULT true,
ADD COLUMN IF NOT EXISTS transfer_time_minutes INTEGER DEFAULT 2,
ADD COLUMN IF NOT EXISTS equipment JSONB DEFAULT '[]',
-- e.g., ["projector", "smartboard", "whiteboard"]
ADD COLUMN IF NOT EXISTS max_concurrent_bookings INTEGER DEFAULT 1;

-- Fix 2: Change timetable_slots.room to proper FK
ALTER TABLE timetable_slots
ADD COLUMN room_id UUID REFERENCES rooms(id);

-- Migrate existing string data
UPDATE timetable_slots ts
SET room_id = (
  SELECT r.id FROM rooms r 
  WHERE r.name = ts.room 
  LIMIT 1
)
WHERE ts.room IS NOT NULL;

-- Drop old column after migration
-- ALTER TABLE timetable_slots DROP COLUMN room;

-- Fix 3: Building transfer matrix
CREATE TABLE building_transfer_times (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  from_building VARCHAR(100) NOT NULL,
  to_building VARCHAR(100) NOT NULL,
  transfer_minutes INTEGER NOT NULL DEFAULT 5,
  is_feasible_consecutive BOOLEAN DEFAULT false,
  -- Can students go between these buildings 
  -- in consecutive periods?
  notes TEXT,
  UNIQUE(school_id, from_building, to_building)
);

-- Fix 4: Room booking tracking
CREATE TABLE room_bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id),
  timetable_slot_id UUID REFERENCES timetable_slots(id),
  day_of_week INTEGER NOT NULL,
  period_id UUID NOT NULL REFERENCES periods(id),
  booked_by_class_id UUID REFERENCES classes(id),
  booked_by_section_id UUID REFERENCES sections(id),
  booking_type VARCHAR(30) DEFAULT 'timetable',
  -- 'timetable', 'exam', 'event', 'maintenance'
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_rooms_building ON rooms(building, floor);
CREATE INDEX idx_rooms_type ON rooms(room_type);
CREATE INDEX idx_room_bookings_room_day 
  ON room_bookings(room_id, day_of_week, period_id);
```

### Enhanced Room Allocator

```javascript
// NEW FILE: room-allocator.js

class RoomAllocator {
  
  constructor(rooms, buildingTransfers, sectionSizes) {
    this.rooms = rooms;
    this.buildingTransfers = buildingTransfers;
    this.sectionSizes = sectionSizes;
    this.bookings = new Map(); // roomId-day-periodId → slotInfo
  }
  
  /**
   * Find best room for a subject assignment
   */
  findRoom(slot, existingSlots, data) {
    const sectionSize = this.sectionSizes[
      `${slot.classId}-${slot.sectionId}`
    ] || 40;
    
    const requiresLab = slot.requiresLab;
    const roomType = requiresLab ? 'lab' : 'classroom';
    const specificLabType = slot.labType; 
    // 'physics_lab', 'chemistry_lab', 'computer_lab', etc.
    
    // Step 1: Get candidate rooms
    let candidates = this.rooms.filter(r => {
      // Basic type match
      if (requiresLab) {
        if (specificLabType && r.room_type !== specificLabType) {
          return false;
        }
        if (!specificLabType && r.room_type !== 'lab') {
          return false;
        }
      }
      
      // Capacity check
      if (r.capacity && r.capacity < sectionSize) return false;
      
      // Availability check
      if (!r.is_available) return false;
      
      return true;
    });
    
    // Step 2: Check room not already booked
    candidates = candidates.filter(r => {
      const key = `${r.id}-${slot.day}-${slot.periodId}`;
      return !this.bookings.has(key);
    });
    
    // Step 3: Check building transfer feasibility
    if (candidates.length > 1) {
      const prevSlot = this.getPreviousSlot(
        slot, existingSlots
      );
      if (prevSlot && prevSlot.roomId) {
        const prevRoom = this.rooms.find(
          r => r.id === prevSlot.roomId
        );
        if (prevRoom) {
          // Prefer same building
          candidates.sort((a, b) => {
            const aTransfer = this.getTransferTime(
              prevRoom.building, a.building
            );
            const bTransfer = this.getTransferTime(
              prevRoom.building, b.building
            );
            return aTransfer - bTransfer;
          });
        }
      }
    }
    
    // Step 4: Check dedicated rooms first
    const dedicatedRoom = candidates.find(
      r => r.class_id === slot.classId && 
           r.section_id === slot.sectionId
    );
    if (dedicatedRoom) return dedicatedRoom;
    
    // Step 5: Return best available
    if (candidates.length === 0) {
      return null; // No room available
    }
    
    // Score candidates
    candidates.sort((a, b) => {
      let scoreA = 0, scoreB = 0;
      
      // Prefer rooms with closer capacity match
      scoreA += Math.abs(
        (a.capacity || 50) - sectionSize
      );
      scoreB += Math.abs(
        (b.capacity || 50) - sectionSize
      );
      
      // Prefer lower floor for accessibility
      scoreA += (a.floor || 0) * 2;
      scoreB += (b.floor || 0) * 2;
      
      // Prefer rooms with required equipment
      if (slot.requiredEquipment) {
        const aEquip = a.equipment || [];
        const bEquip = b.equipment || [];
        const aMissing = slot.requiredEquipment.filter(
          e => !aEquip.includes(e)
        ).length;
        const bMissing = slot.requiredEquipment.filter(
          e => !bEquip.includes(e)
        ).length;
        scoreA += aMissing * 10;
        scoreB += bMissing * 10;
      }
      
      return scoreA - scoreB;
    });
    
    const selected = candidates[0];
    
    // Book the room
    const key = `${selected.id}-${slot.day}-${slot.periodId}`;
    this.bookings.set(key, {
      classId: slot.classId,
      sectionId: slot.sectionId,
      subjectId: slot.subjectId
    });
    
    return selected;
  }
  
  getTransferTime(fromBuilding, toBuilding) {
    if (!fromBuilding || !toBuilding) return 0;
    if (fromBuilding === toBuilding) return 0;
    
    const transfer = this.buildingTransfers.find(
      t => t.from_building === fromBuilding && 
           t.to_building === toBuilding
    );
    return transfer?.transfer_minutes || 5;
  }
}

module.exports = RoomAllocator;
```

---

# 📊 QUESTION 6-7: Substitution & Versioning

## These are 60% complete. Key additions needed:

```sql
-- MIGRATION: 009_enhanced_substitution_versioning.sql

-- ======= SUBSTITUTION ENHANCEMENTS =======

-- Substitute availability tracking
CREATE TABLE substitute_availability (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  teacher_id UUID NOT NULL REFERENCES teachers(id),
  date DATE NOT NULL,
  is_available BOOLEAN DEFAULT true,
  max_substitutions_per_day INTEGER DEFAULT 3,
  preferred_subjects UUID[] DEFAULT NULL,
  -- Subjects they can substitute for
  notes TEXT,
  UNIQUE(teacher_id, date)
);

-- Substitution analytics
CREATE TABLE substitution_analytics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL,
  academic_year_id UUID NOT NULL,
  teacher_id UUID NOT NULL REFERENCES teachers(id),
  month INTEGER NOT NULL,
  year INTEGER NOT NULL,
  total_absences INTEGER DEFAULT 0,
  total_substitutions_given INTEGER DEFAULT 0,
  total_substitutions_received INTEGER DEFAULT 0,
  computed_at TIMESTAMP DEFAULT NOW()
);

-- ======= VERSIONING ENHANCEMENTS =======

-- Slot-level locking
ALTER TABLE timetable_slots
ADD COLUMN is_locked BOOLEAN DEFAULT false,
ADD COLUMN locked_by UUID REFERENCES users(id),
ADD COLUMN locked_at TIMESTAMP,
ADD COLUMN lock_reason TEXT;

-- Version change details
ALTER TABLE timetable_versions
ADD COLUMN change_type VARCHAR(50),
-- 'initial', 'teacher_change', 'partial_regen', 
-- 'full_regen', 'manual_edit'
ADD COLUMN affected_classes UUID[] DEFAULT NULL,
ADD COLUMN affected_teachers UUID[] DEFAULT NULL,
ADD COLUMN change_summary JSONB DEFAULT NULL;

-- Slot change audit log
CREATE TABLE timetable_slot_changes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slot_id UUID NOT NULL REFERENCES timetable_slots(id),
  version_id UUID REFERENCES timetable_versions(id),
  change_type VARCHAR(50) NOT NULL,
  -- 'moved', 'swapped', 'teacher_changed', 'locked', 
  -- 'unlocked', 'deleted', 'added'
  old_value JSONB,
  new_value JSONB,
  changed_by UUID REFERENCES users(id),
  changed_at TIMESTAMP DEFAULT NOW(),
  reason TEXT
);
```

---

# 📊 QUESTION 8-9: Algorithm Architecture

## The BIGGEST Problem: Greedy Algorithm Without Backtracking

### Complete Algorithm Rewrite

```javascript
// NEW FILE: csp-scheduler.js
// CSP = Constraint Satisfaction Problem

class CSPScheduler {
  
  constructor(data, constraintRegistry) {
    this.data = data;
    this.constraints = constraintRegistry;
    this.schedule = [];
    this.attempts = 0;
    this.maxAttempts = 1000000;
    this.bestSolution = null;
    this.bestScore = Infinity;
  }
  
  /**
   * MAIN ENTRY POINT
   * Generates timetable for ALL classes simultaneously
   */
  generate(workloads) {
    console.log('🚀 Starting CSP-based generation...');
    console.log(`📋 Total workloads: ${workloads.length}`);
    
    // PHASE 0: Pre-generation validation
    const preCheck = this.preGenerationCheck(workloads);
    if (!preCheck.feasible) {
      return {
        success: false,
        errors: preCheck.errors,
        message: 'Pre-generation check failed: ' +
          'constraints are unsatisfiable'
      };
    }
    
    // PHASE 1: Schedule JOINED LESSONS first
    // (Most constrained - needs all classes free)
    console.log('📌 Phase 1: Scheduling joined lessons...');
    const joinedResults = this.scheduleJoinedLessons(workloads);
    this.schedule.push(...joinedResults);
    
    // PHASE 2: Schedule CONCURRENT GROUPS
    // (Electives - needs multiple teachers free at same time)
    console.log('📌 Phase 2: Scheduling concurrent groups...');
    const concurrentResults = this.scheduleConcurrentGroups(
      workloads
    );
    this.schedule.push(...concurrentResults);
    
    // PHASE 3: Schedule LAB SUBJECTS
    // (Need specific rooms - limited resources)
    console.log('📌 Phase 3: Scheduling lab subjects...');
    const labResults = this.scheduleLabSubjects(workloads);
    this.schedule.push(...labResults);
    
    // PHASE 4: Schedule DOUBLE/TRIPLE periods
    // (Need consecutive slots - harder to find)
    console.log('📌 Phase 4: Scheduling multi-period blocks...');
    const multiResults = this.scheduleMultiPeriodBlocks(workloads);
    this.schedule.push(...multiResults);
    
    // PHASE 5: Schedule REMAINING subjects (with backtracking)
    console.log('📌 Phase 5: Scheduling remaining subjects...');
    const regularResults = this.scheduleWithBacktracking(
      workloads
    );
    this.schedule.push(...regularResults);
    
    // PHASE 6: Optimization pass
    console.log('📌 Phase 6: Optimization...');
    this.optimizationPass();
    
    // PHASE 7: Validation
    console.log('📌 Phase 7: Final validation...');
    const validation = this.validateFinalSchedule();
    
    const score = this.calculateScore();
    
    console.log(`✅ Generation complete. Score: ${score}`);
    console.log(`   Hard violations: ${validation.hardViolations}`);
    console.log(`   Soft penalties: ${validation.softPenalty}`);
    
    return {
      success: validation.hardViolations === 0,
      schedule: this.schedule,
      score,
      validation,
      stats: {
        totalSlots: this.schedule.length,
        attempts: this.attempts,
        joinedLessons: joinedResults.length,
        concurrentGroups: concurrentResults.length,
        labSlots: labResults.length,
        multiPeriodSlots: multiResults.length,
        regularSlots: regularResults.length
      }
    };
  }
  
  /**
   * Pre-generation feasibility check
   */
  preGenerationCheck(workloads) {
    const errors = [];
    
    // Check 1: Teacher workload vs availability
    const teacherLoads = {};
    for (const wl of workloads) {
      for (const subject of wl.subjects) {
        if (!teacherLoads[subject.teacherId]) {
          teacherLoads[subject.teacherId] = {
            totalPeriods: 0,
            availableSlots: 0,
            teacherName: subject.teacherName
          };
        }
        teacherLoads[subject.teacherId].totalPeriods += 
          subject.weeklyFrequency;
      }
    }
    
    for (const [teacherId, load] of 
      Object.entries(teacherLoads)) {
      const availability = this.data.teacherAvailability
        .filter(a => a.teacher_id === teacherId && a.is_available);
      load.availableSlots = availability.length;
      
      if (load.totalPeriods > load.availableSlots) {
        errors.push({
          type: 'TEACHER_OVERLOAD',
          teacherId,
          teacherName: load.teacherName,
          required: load.totalPeriods,
          available: load.availableSlots,
          message: `Teacher "${load.teacherName}" needs ` +
            `${load.totalPeriods} slots but only ` +
            `${load.availableSlots} available`,
          suggestion: 'Reduce subject assignments or ' +
            'increase teacher availability'
        });
      }
    }
    
    // Check 2: Lab capacity
    const labDemand = {};
    for (const wl of workloads) {
      for (const subject of wl.subjects) {
        if (subject.requiresLab) {
          const labType = subject.labType || 'general_lab';
          if (!labDemand[labType]) labDemand[labType] = 0;
          labDemand[labType] += subject.weeklyFrequency;
        }
      }
    }
    
    for (const [labType, demand] of Object.entries(labDemand)) {
      const labs = this.data.rooms.filter(
        r => r.room_type === labType || 
             (labType === 'general_lab' && r.room_type === 'lab')
      );
      const totalCapacity = labs.length * 
        this.data.daysPerWeek * 
        this.data.teachingPeriodsPerDay;
      
      if (demand > totalCapacity) {
        errors.push({
          type: 'LAB_SHORTAGE',
          labType,
          demand,
          capacity: totalCapacity,
          labCount: labs.length,
          message: `${labType}: Need ${demand} lab periods ` +
            `but only ${totalCapacity} capacity ` +
            `(${labs.length} labs)`,
          suggestion: `Add more ${labType} rooms or ` +
            `reduce lab subject frequency`
        });
      }
    }
    
    return {
      feasible: errors.length === 0,
      errors,
      warnings: []
    };
  }
  
  /**
   * BACKTRACKING SCHEDULER
   * The core algorithm that tries, fails, undoes, and retries
   */
  scheduleWithBacktracking(workloads) {
    // Build all occurrences that still need scheduling
    const unscheduled = this.buildUnscheduledOccurrences(
      workloads
    );
    
    // Sort by constraint tightness (Most Constrained Variable)
    unscheduled.sort((a, b) => {
      // Shared teachers first (most constrained)
      const aTeacherLoad = this.getTeacherRemainingLoad(
        a.teacherId
      );
      const bTeacherLoad = this.getTeacherRemainingLoad(
        b.teacherId
      );
      
      // Teachers with fewer available slots go first
      if (aTeacherLoad !== bTeacherLoad) {
        return aTeacherLoad - bTeacherLoad;
      }
      
      // Heavy subjects first (more constrained)
      if (a.isHeavy !== b.isHeavy) return b.isHeavy ? 1 : -1;
      
      return 0;
    });
    
    // Backtracking search
    const result = this.backtrack(unscheduled, 0, []);
    
    if (!result) {
      console.warn('⚠️ Backtracking failed to find complete ' +
        'solution. Returning best partial solution.');
      return this.bestSolution || [];
    }
    
    return result;
  }
  
  /**
   * Recursive backtracking
   */
  backtrack(occurrences, index, assignments) {
    this.attempts++;
    
    // Check timeout
    if (this.attempts > this.maxAttempts) {
      console.warn(`⚠️ Max attempts reached (${this.maxAttempts})`);
      return null;
    }
    
    // Base case: all occurrences assigned
    if (index >= occurrences.length) {
      const score = this.calculatePartialScore(assignments);
      if (score < this.bestScore) {
        this.bestScore = score;
        this.bestSolution = [...assignments];
      }
      return assignments;
    }
    
    const occurrence = occurrences[index];
    
    // Get domain (possible slots) for this occurrence
    const domain = this.getDomain(occurrence);
    
    // Sort domain by score (Least Constraining Value)
    domain.sort((a, b) => {
      const scoreA = this.constraints.scoreSoft(
        { ...occurrence, ...a }, 
        [...this.schedule, ...assignments], 
        this.data
      ).totalPenalty;
      
      const scoreB = this.constraints.scoreSoft(
        { ...occurrence, ...b }, 
        [...this.schedule, ...assignments], 
        this.data
      ).totalPenalty;
      
      return scoreA - scoreB;
    });
    
    // Try each value in domain
    for (const slot of domain) {
      const assignment = {
        ...occurrence,
        day: slot.day,
        periodId: slot.periodId,
        roomId: slot.roomId
      };
      
      // Check hard constraints
      const hardCheck = this.constraints.validateHard(
        assignment,
        [...this.schedule, ...assignments],
        this.data
      );
      
      if (!hardCheck.valid) continue; // Skip invalid
      
      // Forward checking: will this assignment make 
      // future assignments impossible?
      if (this.forwardCheck(
        occurrences, index + 1, 
        [...assignments, assignment]
      )) {
        // Try this assignment
        assignments.push(assignment);
        
        const result = this.backtrack(
          occurrences, index + 1, assignments
        );
        
        if (result) return result; // Found solution!
        
        // Undo (backtrack)
        assignments.pop();
      }
    }
    
    // No valid assignment found
    return null;
  }
  
  /**
   * Forward checking: verify remaining occurrences 
   * still have valid slots
   */
  forwardCheck(occurrences, fromIndex, currentAssignments) {
    const allSlots = [...this.schedule, ...currentAssignments];
    
    for (let i = fromIndex; i < occurrences.length; i++) {
      const occ = occurrences[i];
      const domain = this.getDomain(occ);
      
      // Check if at least ONE slot is valid
      const hasValidSlot = domain.some(slot => {
        const testAssignment = {
          ...occ, 
          day: slot.day, 
          periodId: slot.periodId
        };
        return this.constraints.validateHard(
          testAssignment, allSlots, this.data
        ).valid;
      });
      
      if (!hasValidSlot) return false; // Dead end!
    }
    
    return true;
  }
  
  /**
   * Get all possible slots for an occurrence
   */
  getDomain(occurrence) {
    const slots = [];
    
    const structure = this.data.structures[
      occurrence.structureId
    ];
    const periods = structure?.periods || this.data.teachingPeriods;
    const daysPerWeek = structure?.working_days_per_week || 
      this.data.daysPerWeek;
    
    for (let day = 1; day <= daysPerWeek; day++) {
      for (const period of periods) {
        if (!period.is_teaching) continue;
        
        // Quick pre-filter before expensive constraint checks
        // Teacher available?
        if (!this.isTeacherQuickAvailable(
          occurrence.teacherId, day, period.id
        )) continue;
        
        // Find available room
        const room = this.data.roomAllocator?.findRoom(
          { ...occurrence, day, periodId: period.id },
          this.schedule,
          this.data
        );
        
        slots.push({
          day,
          periodId: period.id,
          periodSequence: period.sequence,
          roomId: room?.id || null
        });
      }
    }
    
    return slots;
  }
  
  /**
   * OPTIMIZATION PASS
   * After initial scheduling, try to improve score
   * by swapping slots
   */
  optimizationPass() {
    console.log('🔧 Starting optimization...');
    
    const maxIterations = 10000;
    let improved = true;
    let iteration = 0;
    let currentScore = this.calculateScore();
    
    while (improved && iteration < maxIterations) {
      improved = false;
      iteration++;
      
      // Try random swaps
      for (let i = 0; i < this.schedule.length; i++) {
        for (let j = i + 1; j < this.schedule.length; j++) {
          const slotA = this.schedule[i];
          const slotB = this.schedule[j];
          
          // Only swap if same class-section
          if (slotA.classId !== slotB.classId || 
              slotA.sectionId !== slotB.sectionId) continue;
          
          // Don't swap locked slots
          if (slotA.isLocked || slotB.isLocked) continue;
          
          // Try swap
          const newA = { 
            ...slotA, 
            day: slotB.day, 
            periodId: slotB.periodId 
          };
          const newB = { 
            ...slotB, 
            day: slotA.day, 
            periodId: slotA.periodId 
          };
          
          // Validate both swapped positions
          const tempSchedule = this.schedule.filter(
            (s, idx) => idx !== i && idx !== j
          );
          
          const validA = this.constraints.validateHard(
            newA, tempSchedule, this.data
          ).valid;
          const validB = this.constraints.validateHard(
            newB, [...tempSchedule, newA], this.data
          ).valid;
          
          if (!validA || !validB) continue;
          
          // Check if swap improves score
          const newSchedule = [...tempSchedule, newA, newB];
          const newScore = this.calculateScoreForSchedule(
            newSchedule
          );
          
          if (newScore < currentScore) {
            // Accept swap
            this.schedule[i] = newA;
            this.schedule[j] = newB;
            currentScore = newScore;
            improved = true;
            break;
          }
        }
        if (improved) break; // Restart outer loop
      }
    }
    
    console.log(`🔧 Optimization: ${iteration} iterations, ` +
      `score ${currentScore}`);
  }
  
  /**
   * Calculate overall schedule score
   * Lower = better
   */
  calculateScore() {
    return this.calculateScoreForSchedule(this.schedule);
  }
  
  calculateScoreForSchedule(schedule) {
    let totalPenalty = 0;
    
    for (const slot of schedule) {
      const { totalPenalty: penalty } = this.constraints.scoreSoft(
        slot, schedule, this.data
      );
      totalPenalty += penalty;
    }
    
    return totalPenalty;
  }
}

module.exports = CSPScheduler;
```

---

# 📊 QUESTION 10: Post-Generation Editing & Export

### Manual Editing API

```javascript
// NEW FILE: slot-editor.controller.js

class SlotEditorController {
  
  /**
   * Move a slot to a new position
   */
  async moveSlot(req, res, next) {
    try {
      const schema = req.tenant?.schema;
      const { slotId } = req.params;
      const { newDay, newPeriodId, validateConflicts } = req.body;
      const userId = req.user?.id;
      
      // Get current slot
      const current = await pool.query(`
        SELECT ts.*, t.class_id, t.section_id 
        FROM ${schema}.timetable_slots ts
        JOIN ${schema}.timetables t ON ts.timetable_id = t.id
        WHERE ts.id = $1
      `, [slotId]);
      
      if (current.rows.length === 0) {
        return res.status(404).json({ 
          success: false, error: 'Slot not found' 
        });
      }
      
      const slot = current.rows[0];
      
      // Check if locked
      if (slot.is_locked) {
        return res.status(403).json({
          success: false,
          error: 'Slot is locked. Unlock before editing.',
          lockedBy: slot.locked_by,
          lockedAt: slot.locked_at,
          lockReason: slot.lock_reason
        });
      }
      
      // Validate conflicts
      if (validateConflicts !== false) {
        const conflicts = await this.checkConflicts(
          schema, slot, newDay, newPeriodId, slotId
        );
        
        if (conflicts.length > 0) {
          return res.status(409).json({
            success: false,
            error: 'Conflicts detected',
            conflicts
          });
        }
      }
      
      // Perform move
      const client = await pool.connect();
      try {
        await client.query('BEGIN');
        
        // Update slot
        await client.query(`
          UPDATE ${schema}.timetable_slots 
          SET day_of_week = $1, 
              period_id = $2, 
              updated_at = NOW()
          WHERE id = $3
        `, [newDay, newPeriodId, slotId]);
        
        // Log change
        await client.query(`
          INSERT INTO ${schema}.timetable_slot_changes 
          (slot_id, change_type, old_value, new_value, 
           changed_by, reason)
          VALUES ($1, 'moved', $2, $3, $4, 'Manual edit')
        `, [
          slotId,
          JSON.stringify({ 
            day: slot.day_of_week, 
            period: slot.period_id 
          }),
          JSON.stringify({ day: newDay, period: newPeriodId }),
          userId
        ]);
        
        await client.query('COMMIT');
        
        res.json({ 
          success: true, 
          message: 'Slot moved successfully' 
        });
        
      } catch (err) {
        await client.query('ROLLBACK');
        throw err;
      } finally {
        client.release();
      }
      
    } catch (error) {
      next(error);
    }
  }
  
  /**
   * Swap two slots
   */
  async swapSlots(req, res, next) {
    try {
      const schema = req.tenant?.schema;
      const { slot1Id, slot2Id } = req.body;
      const userId = req.user?.id;
      
      const client = await pool.connect();
      
      try {
        await client.query('BEGIN');
        
        const s1 = (await client.query(`
          SELECT * FROM ${schema}.timetable_slots WHERE id = $1
        `, [slot1Id])).rows[0];
        
        const s2 = (await client.query(`
          SELECT * FROM ${schema}.timetable_slots WHERE id = $1
        `, [slot2Id])).rows[0];
        
        if (!s1 || !s2) throw new Error('Slots not found');
        if (s1.is_locked || s2.is_locked) {
          throw new Error('Cannot swap locked slots');
        }
        
        // Swap positions
        await client.query(`
          UPDATE ${schema}.timetable_slots 
          SET day_of_week = $1, period_id = $2, updated_at = NOW()
          WHERE id = $3
        `, [s2.day_of_week, s2.period_id, slot1Id]);
        
        await client.query(`
          UPDATE ${schema}.timetable_slots 
          SET day_of_week = $1, period_id = $2, updated_at = NOW()
          WHERE id = $3
        `, [s1.day_of_week, s1.period_id, slot2Id]);
        
        // Log both changes
        await client.query(`
          INSERT INTO ${schema}.timetable_slot_changes 
          (slot_id, change_type, old_value, new_value, changed_by)
          VALUES 
          ($1, 'swapped', $2, $3, $4),
          ($5, 'swapped', $6, $7, $8)
        `, [
          slot1Id,
          JSON.stringify({ 
            day: s1.day_of_week, period: s1.period_id 
          }),
          JSON.stringify({ 
            day: s2.day_of_week, period: s2.period_id 
          }),
          userId,
          slot2Id,
          JSON.stringify({ 
            day: s2.day_of_week, period: s2.period_id 
          }),
          JSON.stringify({ 
            day: s1.day_of_week, period: s1.period_id 
          }),
          userId
        ]);
        
        await client.query('COMMIT');
        res.json({ success: true, message: 'Slots swapped' });
        
      } catch (err) {
        await client.query('ROLLBACK');
        throw err;
      } finally {
        client.release();
      }
      
    } catch (error) {
      next(error);
    }
  }
  
  /**
   * Lock/Unlock slots
   */
  async lockSlot(req, res, next) {
    const { slotId } = req.params;
    const { reason } = req.body;
    const userId = req.user?.id;
    const schema = req.tenant?.schema;
    
    await pool.query(`
      UPDATE ${schema}.timetable_slots 
      SET is_locked = true, 
          locked_by = $1, 
          locked_at = NOW(), 
          lock_reason = $2
      WHERE id = $3
    `, [userId, reason, slotId]);
    
    res.json({ success: true, message: 'Slot locked' });
  }
  
  async unlockSlot(req, res, next) {
    const { slotId } = req.params;
    const schema = req.tenant?.schema;
    
    await pool.query(`
      UPDATE ${schema}.timetable_slots 
      SET is_locked = false, 
          locked_by = NULL, 
          locked_at = NULL, 
          lock_reason = NULL
      WHERE id = $1
    `, [slotId]);
    
    res.json({ success: true, message: 'Slot unlocked' });
  }
  
  /**
   * Bulk lock by subject
   */
  async lockBySubject(req, res, next) {
    const { subjectId, reason } = req.body;
    const userId = req.user?.id;
    const schema = req.tenant?.schema;
    
    const result = await pool.query(`
      UPDATE ${schema}.timetable_slots 
      SET is_locked = true, 
          locked_by = $1, 
          locked_at = NOW(), 
          lock_reason = $2
      WHERE subject_id = $3 AND is_active = true
      RETURNING id
    `, [userId, reason, subjectId]);
    
    res.json({ 
      success: true, 
      message: `Locked ${result.rowCount} slots` 
    });
  }
}

module.exports = new SlotEditorController();
```

### PDF Export Service

```javascript
// NEW FILE: export.service.js

const PDFDocument = require('pdfkit');
const ExcelJS = require('exceljs');

class ExportService {
  
  /**
   * Generate Class-wise PDF Timetable
   */
  async generateClassPDF(schema, runId, classId) {
    const slots = await this.getSlots(schema, runId, classId);
    const periods = await this.getPeriods(schema);
    const days = ['Monday', 'Tuesday', 'Wednesday', 
      'Thursday', 'Friday', 'Saturday'];
    
    const doc = new PDFDocument({ 
      size: 'A4', 
      layout: 'landscape',
      margin: 30 
    });
    
    // Header
    doc.fontSize(18).font('Helvetica-Bold')
      .text(`Timetable - ${slots[0]?.class_name || 'Class'} ` +
        `${slots[0]?.section_name || ''}`, { align: 'center' });
    doc.moveDown(0.5);
    
    // Build grid
    const teachingPeriods = periods.filter(
      p => p.is_teaching !== false
    );
    const allPeriods = periods.sort(
      (a, b) => a.sequence - b.sequence
    );
    
    // Table dimensions
    const tableLeft = 30;
    const tableTop = 100;
    const colWidth = 110;
    const rowHeight = 50;
    const headerColWidth = 80;
    
    // Draw header row (days)
    doc.fontSize(9).font('Helvetica-Bold');
    doc.rect(tableLeft, tableTop, headerColWidth, rowHeight)
      .fillAndStroke('#2563EB', '#1E40AF');
    doc.fillColor('white')
      .text('Period', tableLeft + 5, tableTop + 18, {
        width: headerColWidth - 10, align: 'center'
      });
    
    for (let d = 0; d < days.length; d++) {
      const x = tableLeft + headerColWidth + (d * colWidth);
      doc.rect(x, tableTop, colWidth, rowHeight)
        .fillAndStroke('#2563EB', '#1E40AF');
      doc.fillColor('white')
        .text(days[d], x + 5, tableTop + 18, {
          width: colWidth - 10, align: 'center'
        });
    }
    
    // Draw period rows
    for (let p = 0; p < allPeriods.length; p++) {
      const period = allPeriods[p];
      const y = tableTop + rowHeight + (p * rowHeight);
      
      // Period label
      const bgColor = period.is_break ? '#FEF3C7' : '#F8FAFC';
      doc.rect(tableLeft, y, headerColWidth, rowHeight)
        .fillAndStroke(bgColor, '#CBD5E1');
      doc.fillColor('#1E293B').font('Helvetica-Bold')
        .fontSize(8)
        .text(period.name, tableLeft + 3, y + 5, {
          width: headerColWidth - 6
        });
      doc.font('Helvetica').fontSize(7)
        .text(
          `${period.start_time}-${period.end_time}`, 
          tableLeft + 3, y + 20, {
            width: headerColWidth - 6
          }
        );
      
      // Day cells
      for (let d = 0; d < days.length; d++) {
        const x = tableLeft + headerColWidth + (d * colWidth);
        
        if (period.is_break) {
          doc.rect(x, y, colWidth, rowHeight)
            .fillAndStroke('#FEF3C7', '#CBD5E1');
          doc.fillColor('#92400E').font('Helvetica-Oblique')
            .fontSize(9)
            .text(period.name, x + 5, y + 18, {
              width: colWidth - 10, align: 'center'
            });
        } else {
          const slot = slots.find(
            s => s.day_of_week === (d + 1) && 
                 s.period_id === period.id
          );
          
          if (slot) {
            // Subject color based on type
            const subjectColor = this.getSubjectColor(
              slot.subject_category
            );
            doc.rect(x, y, colWidth, rowHeight)
              .fillAndStroke(subjectColor.bg, '#CBD5E1');
            doc.fillColor(subjectColor.text)
              .font('Helvetica-Bold').fontSize(9)
              .text(slot.subject_name, x + 3, y + 5, {
                width: colWidth - 6
              });
            doc.font('Helvetica').fontSize(7)
              .text(slot.teacher_name, x + 3, y + 22, {
                width: colWidth - 6
              });
            if (slot.room_name) {
              doc.text(
                `📍 ${slot.room_name}`, x + 3, y + 35, {
                  width: colWidth - 6
                }
              );
            }
          } else {
            doc.rect(x, y, colWidth, rowHeight)
              .fillAndStroke('#F8FAFC', '#CBD5E1');
            doc.fillColor('#94A3B8').fontSize(8)
              .text('—', x + colWidth/2 - 5, y + 18);
          }
        }
      }
    }
    
    return doc;
  }
  
  getSubjectColor(category) {
    const colors = {
      'heavy': { bg: '#DBEAFE', text: '#1E40AF' },
      'medium': { bg: '#D1FAE5', text: '#065F46' },
      'light': { bg: '#FEF3C7', text: '#92400E' },
      'practical': { bg: '#E0E7FF', text: '#3730A3' },
      'physical': { bg: '#FCE7F3', text: '#9D174D' },
      'default': { bg: '#F1F5F9', text: '#334155' }
    };
    return colors[category] || colors.default;
  }
}

module.exports = new ExportService();
```

### New API Routes

```javascript
// ADD TO: timetable-generator.routes.js

// ========== EDITING ==========
router.put(
  '/slots/:slotId/move', 
  auth, roleCheck(['admin', 'principal']),
  slotEditor.moveSlot
);
router.post(
  '/slots/swap', 
  auth, roleCheck(['admin', 'principal']),
  slotEditor.swapSlots
);
router.post(
  '/slots/:slotId/lock', 
  auth, roleCheck(['admin', 'principal']),
  slotEditor.lockSlot
);
router.post(
  '/slots/:slotId/unlock', 
  auth, roleCheck(['admin', 'principal']),
  slotEditor.unlockSlot
);
router.post(
  '/slots/lock-by-subject', 
  auth, roleCheck(['admin', 'principal']),
  slotEditor.lockBySubject
);

// ========== EXPORT ==========
router.get(
  '/export/pdf/class/:runId/:classId', 
  auth, 
  exportController.exportClassPDF
);
router.get(
  '/export/pdf/teacher/:runId/:teacherId', 
  auth, 
  exportController.exportTeacherPDF
);
router.get(
  '/export/pdf/room/:runId/:roomId', 
  auth, 
  exportController.exportRoomPDF
);
router.get(
  '/export/excel/:runId', 
  auth, 
  exportController.exportExcel
);

// ========== LESSON GRID ==========
router.get(
  '/runs/:runId/lesson-grid', 
  auth, 
  controller.getLessonGrid
);

// ========== CONSTRAINT CONFIG ==========
router.get(
  '/constraints', 
  auth, 
  controller.getConstraintConfig
);
router.put(
  '/constraints/:constraintId', 
  auth, 
  controller.updateConstraintConfig
);

// ========== STUDENT GROUPS ==========
router.get('/student-groups', auth, groupController.list);
router.post('/student-groups', auth, groupController.create);
router.put('/student-groups/:id', auth, groupController.update);
router.delete('/student-groups/:id', auth, groupController.delete);

// ========== STREAMS ==========
router.get('/streams', auth, streamController.list);
router.post('/streams', auth, streamController.create);

// ========== JOINED LESSONS ==========
router.get('/joined-lessons', auth, joinedController.list);
router.post('/joined-lessons', auth, joinedController.create);
```

---

# 📊 MASTER IMPLEMENTATION PRIORITY

```
🔴 PHASE 1: CRITICAL (Weeks 1-3)
├── Q2: Class Divisions, Groups & Electives
│   ├── Database tables (student_groups, concurrent_sets, etc.)
│   ├── Concurrent group scheduler
│   └── API endpoints
├── Q8: Algorithm Upgrade (CSP with Backtracking)
│   ├── ConstraintRegistry system
│   ├── CSP Scheduler with backtracking
│   └── Pre-generation validation
└── Q9: Whole-School Generation
    ├── Phased generation (joined→concurrent→labs→regular)
    └── Global optimization pass

🟡 PHASE 2: HIGH PRIORITY (Weeks 4-6)
├── Q3: Full Constraint Engine
│   ├── 15+ constraint implementations
│   ├── School-level configuration
│   └── Hard/Soft differentiation
├── Q1: Double/Triple Period Enhancement
│   ├── lesson_pattern column & parser
│   ├── Consecutive slot finder
│   └── Validation
└── Q10: Manual Editing
    ├── Move/Swap/Lock APIs
    ├── Conflict detection on edit
    └── Change audit trail

🟢 PHASE 3: IMPORTANT (Weeks 7-9)
├── Q5: Room System Enhancement
│   ├── Building/Floor columns
│   ├── Capacity validation
│   └── Transfer time checking
├── Q10: Export System
│   ├── PDF generation (class/teacher/room)
│   ├── Excel export
│   └── Lesson grid view
└── Q7: Versioning Enhancement
    ├── Slot-level locking
    ├── Version comparison
    └── Teacher replacement API

🔵 PHASE 4: POLISH (Weeks 10-12)
├── Q6: Substitution Enhancement
│   ├── Substitute availability calendar
│   ├── Auto-notification system
│   └── Analytics dashboard
├── Performance Optimization
│   ├── Caching strategy
│   ├── Parallel generation
│   └── Progress tracking WebSocket
└── Testing & Documentation
    ├── Unit tests for all constraints
    ├── Integration tests
    └── User documentation
```

---

# 📈 SCORE IMPROVEMENT PROJECTION

```
CURRENT SYSTEM SCORES:
├── Double/Triple Periods:    85% → Target: 100%
├── Class Divisions/Groups:    0% → Target: 95%
├── Constraint Engine:        47% → Target: 90%
├── Grade-Specific Periods:  100% → Stays: 100%
├── Room Management:          40% → Target: 90%
├── Substitution:             60% → Target: 85%
├── Versioning:               60% → Target: 90%
├── Algorithm Quality:     4/10 → Target: 8/10
├── Whole-School Generation:  ✅ → Enhanced: ✅+
└── Post-Gen Editing:         25% → Target: 85%

OVERALL: 42% → Target: 90%+ (Production ASC-equivalent)
```

This is the **complete implementation plan**. Each phase builds on the previous one. The most critical change is **Q2 (Groups/Divisions) + Q8 (CSP Algorithm)** — these unlock everything else. Want me to deep-dive into any specific phase?
