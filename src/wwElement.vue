<template>
  <div class="course-distribution">
    <!-- Header -->
    <div class="course-distribution__header">
      <div class="course-distribution__header-left">
        <h1 class="course-distribution__title">Course Enrollment</h1>
        <h2 class="course-distribution__subtitle">
          {{ currentSemester || '—' }} – {{ selectedDayName }}
        </h2>
      </div>
      <div class="course-distribution__header-right">
        <!-- Reset to First Choices Button -->
        <button
          @click="onResetToFirstChoices"
          class="course-distribution__reset-btn"
          title="Reset all students to their first choice preferences"
        >
          <svg class="course-distribution__reset-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
          </svg>
          Reset to First Choices
        </button>

        <div class="course-distribution__search">
          <input
            type="text"
            placeholder="Search students..."
            v-model="searchQuery"
            @input="onSearchChange($event.target.value)"
            class="course-distribution__search-input"
          />
        </div>

        <!-- Semester Selector -->
        <div class="course-distribution__semester-selector">
          <select
            :value="selectedSemesterIndex"
            @change="onSemesterChange($event.target.value)"
            class="course-distribution__semester-select"
          >
            <option
              v-for="(semester, index) in semesters"
              :key="semester.id"
              :value="index"
            >
              {{ semester.name }}
            </option>
          </select>
        </div>

        <div class="course-distribution__day-selector">
          <select
            :value="selectedDayNumber"
            @change="onDayChange($event.target.value)"
            class="course-distribution__day-select"
          >
            <option
              v-for="day in daysOfWeek"
              :key="day.day_id"
              :value="day.day_number"
            >
              {{ day.name_en }}
            </option>
          </select>
        </div>
      </div>
    </div>

    <div v-if="loading" class="course-distribution__loading">Loading...</div>

    <div v-else-if="filteredCourses.length === 0 && totalEnrolledForDay === 0" class="course-distribution__empty">
      <p>No courses or students available for the selected criteria.</p>
    </div>

    <div v-else class="course-distribution__content">
      <!-- Problems Section -->
      <div v-if="unresolvedProblems.length > 0" class="course-distribution__problems">
        <h3 class="course-distribution__problems-title">
          <svg class="course-distribution__problems-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.732-.833-2.464 0L4.35 16.5c-.77.833.192 2.5 1.732 2.5z" />
          </svg>
          Problems Requiring Resolution ({{ unresolvedProblems.length }})
        </h3>
        <div class="course-distribution__problems-list">
          <div 
            v-for="note in unresolvedProblems" 
            :key="note.id" 
            class="course-distribution__problems-item"
          >
            <div class="course-distribution__problems-content">
              <div class="course-distribution__problems-header">
                <span class="course-distribution__problems-course">{{ getCourseName(note.course_id) }}</span>
                <span class="course-distribution__problems-author">{{ note.author }}</span>
                <span class="course-distribution__problems-date">{{ formatDate(note.created_at) }}</span>
              </div>
              <p class="course-distribution__problems-text">{{ note.text }}</p>
            </div>
            <button
              @click="onResolveNote(note.id)"
              class="course-distribution__problems-resolve-btn"
            >
              <svg class="course-distribution__problems-resolve-icon" fill="currentColor" viewBox="0 0 20 20">
                <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
              </svg>
              Resolve
            </button>
          </div>
        </div>
      </div>

      <!-- Waiting List -->
      <div v-if="waitingListStudents.length > 0" class="course-distribution__waiting-list">
        <div class="course-distribution__waiting-list-header">
          <h3 class="course-distribution__waiting-list-title">
            <svg class="course-distribution__waiting-list-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4.354a4 4 0 110 5.292M15 21H3v-1a6 6 0 0112 0v1zm0 0h6v-1a6 6 0 00-9-5.197m13.5-9a2.5 2.5 0 11-5 0 2.5 2.5 0 015 0z" />
            </svg>
            Waiting ({{ waitingListStudents.length }})
          </h3>
        </div>

        <div class="course-distribution__waiting-list-content">
          <!-- Header row -->
          <div class="course-distribution__waiting-list-header-row">
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--participants">Participants</div>
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wish">1. wish</div>
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wish">2. wish</div>
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wish">3. wish</div>
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wait">Wait</div>
          </div>

          <!-- Student rows -->
          <div v-for="student in waitingListStudents" :key="student.id + '-' + selectedDayNumber" class="course-distribution__waiting-list-row">
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--participants">
              <span class="course-distribution__waiting-list-student-name">{{ student.class }} {{ student.name }}</span>
              <div class="course-distribution__waiting-list-student-schedule">
                <span 
                  v-for="(enrollment, dayIndex) in student.enrollments" 
                  :key="dayIndex"
                  class="course-distribution__waiting-list-schedule-chip"
                >
                  {{ getDayName(dayIndex) }} ({{ parseInt(dayIndex) + 1 }}): {{ enrollment || 'Home' }}
                </span>
              </div>
            </div>

                                                   <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wish">
                <button
                  v-if="student.first_choice"
                  class="course-distribution__waiting-list-preference-btn"
                  :class="[
                    isCurrentChoice(student, 1) ? 'course-distribution__waiting-list-preference-btn--first-choice' : '',
                    student.first_choice === 'go-home' ? 'course-distribution__waiting-list-preference-btn--go-home' : '',
                    isLockedTarget(student.first_choice) ? 'course-distribution__waiting-list-preference-btn--locked' : ''
                  ]"
                  :disabled="isCurrentChoice(student, 1) || isLockedTarget(student.first_choice)"
                  @click="!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice) && onStudentMove(student.id, student.first_choice)"
                  :title="isLockedTarget(student.first_choice) ? 'Course locked' : formatPreferenceDisplay(student.first_choice)"
                >
                  {{ formatPreferenceDisplay(student.first_choice) }}
                </button>
              </div>

                                                   <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wish">
                <button
                  v-if="student.second_choice"
                  class="course-distribution__waiting-list-preference-btn"
                  :class="[
                    isCurrentChoice(student, 2) ? 'course-distribution__waiting-list-preference-btn--second-choice' : '',
                    student.second_choice === 'go-home' ? 'course-distribution__waiting-list-preference-btn--go-home' : '',
                    isLockedTarget(student.second_choice) ? 'course-distribution__waiting-list-preference-btn--locked' : ''
                  ]"
                  :disabled="isCurrentChoice(student, 2) || isLockedTarget(student.second_choice)"
                  @click="!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice) && onStudentMove(student.id, student.second_choice)"
                  :title="isLockedTarget(student.second_choice) ? 'Course locked' : formatPreferenceDisplay(student.second_choice)"
                >
                  {{ formatPreferenceDisplay(student.second_choice) }}
                </button>
              </div>

                                                   <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wish">
                <button
                  v-if="student.third_choice"
                  class="course-distribution__waiting-list-preference-btn"
                  :class="[
                    isCurrentChoice(student, 3) ? 'course-distribution__waiting-list-preference-btn--third-choice' : '',
                    student.third_choice === 'go-home' ? 'course-distribution__waiting-list-preference-btn--go-home' : '',
                    isLockedTarget(student.third_choice) ? 'course-distribution__waiting-list-preference-btn--locked' : ''
                  ]"
                  :disabled="isCurrentChoice(student, 3) || isLockedTarget(student.third_choice)"
                  @click="!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice) && onStudentMove(student.id, student.third_choice)"
                  :title="isLockedTarget(student.third_choice) ? 'Course locked' : formatPreferenceDisplay(student.third_choice)"
                >
                  {{ formatPreferenceDisplay(student.third_choice) }}
                </button>
              </div>

            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--wait">
              <button
                v-if="!hasGoHomePreference(student)"
                class="course-distribution__waiting-list-wait-btn"
                title="Move to waiting list"
                @click="onStudentMove(student.id, 'waiting')"
              >
                W
              </button>
              <div v-else class="course-distribution__waiting-list-wait-placeholder"></div>
            </div>
          </div>
        </div>
      </div>

            <p v-if="waitingListStudents.length === 0" class="course-distribution__waiting-list-empty">No students waiting</p>

      <!-- Going Home Section -->
      <div class="course-distribution__going-home">
        <div 
          class="course-distribution__going-home-header"
          @click="isGoingHomeExpanded = !isGoingHomeExpanded"
        >
          <svg class="course-distribution__going-home-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6" />
          </svg>
          <h3 class="course-distribution__going-home-title">
            Going Home at 14:30
          </h3>
          <span class="course-distribution__going-home-badge">
            {{ goingHomeStudents.length }}/{{ totalStudentsInSchool }}
          </span>
          <svg 
            class="course-distribution__going-home-arrow" 
            :class="{ 'course-distribution__going-home-arrow--expanded': isGoingHomeExpanded }"
            fill="none" viewBox="0 0 24 24" stroke="currentColor"
          >
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
          </svg>
        </div>
        
        <div v-if="isGoingHomeExpanded" class="course-distribution__going-home-content">
          <!-- First Choice Go Home - Simple List -->
          <div v-if="firstChoiceGoHomeStudents.length > 0" class="course-distribution__going-home-section">
            <h4 class="course-distribution__going-home-section-title">Going Home as First Choice</h4>
            <div class="course-distribution__going-home-classes">
              <div 
                v-for="classGroup in firstChoiceGoHomeClassStats" 
                :key="classGroup.className" 
                class="course-distribution__going-home-class"
              >
                <div class="course-distribution__going-home-class-header">
                  <span class="course-distribution__going-home-class-name">{{ classGroup.className }}</span>
                  <span class="course-distribution__going-home-class-badge">
                    {{ classGroup.goingHome }}/{{ classGroup.total }}
                  </span>
                </div>
                <div class="course-distribution__going-home-students">
                  <div 
                    v-for="student in classGroup.children" 
                    :key="student.id"
                    class="course-distribution__going-home-student"
                  >
                    <span class="course-distribution__going-home-student-name">{{ student.name }}</span>
                    <div class="course-distribution__going-home-student-schedule">
                      <span 
                        v-for="(enrollment, dayIndex) in student.enrollments" 
                        :key="dayIndex"
                        class="course-distribution__going-home-schedule-chip"
                      >
                        {{ getDayName(dayIndex) }} ({{ parseInt(dayIndex) + 1 }}): {{ enrollment || 'Home' }}
                      </span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Second/Third Choice Go Home - Grid Format -->
          <div v-if="secondOrThirdChoiceGoHomeStudents.length > 0" class="course-distribution__going-home-secondary">
            <div class="course-distribution__going-home-secondary-header">
              <h4 class="course-distribution__going-home-secondary-title">
                <span>Going Home as 2nd/3rd Choice</span>
                <span class="course-distribution__going-home-secondary-badge">
                  {{ secondOrThirdChoiceGoHomeStudents.length }} children - can be reassigned
                </span>
              </h4>
            </div>

            <div v-if="isGoingHomeChildrenVisible" class="course-distribution__going-home-grid">
              <!-- Grid Header -->
              <div class="course-distribution__going-home-grid-header">
                <div class="course-distribution__going-home-grid-columns">
                  <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--participants">Participants</div>
                  <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wish">1. wish</div>
                  <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wish">2. wish</div>
                  <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wish">3. wish</div>
                  <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wait">Wait</div>
                </div>
                <div class="course-distribution__going-home-toggle">
                  <button
                    class="course-distribution__going-home-toggle-btn"
                    @click="isGoingHomeChildrenVisible = !isGoingHomeChildrenVisible"
                    :title="isGoingHomeChildrenVisible ? 'Hide participants' : 'Show participants'"
                  >
                    <svg v-if="isGoingHomeChildrenVisible" class="course-distribution__going-home-toggle-icon course-distribution__going-home-toggle-icon--up" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 15l7-7 7 7" />
                    </svg>
                    <svg v-else class="course-distribution__going-home-toggle-icon course-distribution__going-home-toggle-icon--down" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                    </svg>
                  </button>
                </div>
              </div>

              <div v-if="secondOrThirdChoiceGoHomeStudents.length > 0" class="course-distribution__going-home-grid-rows">
                <div 
                  v-for="student in secondOrThirdChoiceGoHomeStudents" 
                  :key="student.id"
                  class="course-distribution__going-home-grid-row"
                >
                  <div class="course-distribution__going-home-grid-columns">
                    <!-- Participants Column -->
                    <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--participants">
                      <span class="course-distribution__going-home-student-info">{{ student.class }} {{ student.name }}</span>
                      <div class="course-distribution__going-home-student-schedule">
                        <span 
                          v-for="(enrollment, dayIndex) in student.enrollments" 
                          :key="dayIndex"
                          class="course-distribution__going-home-schedule-chip"
                        >
                          {{ getDayName(dayIndex) }} ({{ parseInt(dayIndex) + 1 }}): {{ enrollment || 'Home' }}
                        </span>
                      </div>
                    </div>

                                         <!-- First Choice Column -->
                                           <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wish">
                        <button
                          v-if="student.first_choice"
                          class="course-distribution__going-home-preference-btn"
                          :class="[
                            isCurrentChoice(student, 1) ? 'course-distribution__going-home-preference-btn--first-choice' : '',
                            student.first_choice === 'go-home' ? 'course-distribution__going-home-preference-btn--go-home' : ''
                          ]"
                          :disabled="isCurrentChoice(student, 1)"
                          @click="!isCurrentChoice(student, 1) && onStudentMove(student.id, student.first_choice)"
                          :title="formatPreferenceDisplay(student.first_choice)"
                        >
                          {{ formatPreferenceDisplay(student.first_choice) }}
                        </button>
                      </div>
                    
                                         <!-- Second Choice Column -->
                                           <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wish">
                        <button
                          v-if="student.second_choice"
                          class="course-distribution__going-home-preference-btn"
                          :class="[
                            isCurrentChoice(student, 2) ? 'course-distribution__going-home-preference-btn--second-choice' : '',
                            student.second_choice === 'go-home' ? 'course-distribution__going-home-preference-btn--go-home' : ''
                          ]"
                          :disabled="isCurrentChoice(student, 2)"
                          @click="!isCurrentChoice(student, 2) && onStudentMove(student.id, student.second_choice)"
                          :title="formatPreferenceDisplay(student.second_choice)"
                        >
                          {{ formatPreferenceDisplay(student.second_choice) }}
                        </button>
                      </div>
                    
                                         <!-- Third Choice Column -->
                                           <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wish">
                        <button
                          v-if="student.third_choice"
                          class="course-distribution__going-home-preference-btn"
                          :class="[
                            isCurrentChoice(student, 3) ? 'course-distribution__going-home-preference-btn--third-choice' : '',
                            student.third_choice === 'go-home' ? 'course-distribution__going-home-preference-btn--go-home' : ''
                          ]"
                          :disabled="isCurrentChoice(student, 3)"
                          @click="!isCurrentChoice(student, 3) && onStudentMove(student.id, student.third_choice)"
                          :title="formatPreferenceDisplay(student.third_choice)"
                        >
                          {{ formatPreferenceDisplay(student.third_choice) }}
                        </button>
                      </div>

                    <!-- Waiting List Column -->
                    <div class="course-distribution__going-home-grid-col course-distribution__going-home-grid-col--wait">
                      <button
                        v-if="!hasGoHomePreference(student)"
                        class="course-distribution__going-home-wait-btn"
                        @click="onStudentMove(student.id, 'waiting')"
                        title="Move to waiting list"
                      >
                        W
                      </button>
                      <div v-else class="course-distribution__going-home-wait-placeholder"></div>
                    </div>
                  </div>
                  <div class="course-distribution__going-home-grid-spacer">
                    <!-- Empty space to match the toggle button area in the header -->
                  </div>
                </div>
              </div>
            </div>

            <!-- Show toggle button when participants are hidden -->
            <div v-if="!isGoingHomeChildrenVisible" class="course-distribution__going-home-toggle-hidden">
              <button
                class="course-distribution__going-home-toggle-btn"
                @click="isGoingHomeChildrenVisible = !isGoingHomeChildrenVisible"
                title="Show participants"
              >
                <svg class="course-distribution__going-home-toggle-icon course-distribution__going-home-toggle-icon--down" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                </svg>
              </button>
            </div>
          </div>

          <div v-if="firstChoiceGoHomeStudents.length === 0 && secondOrThirdChoiceGoHomeStudents.length === 0" class="course-distribution__going-home-empty">
            {{ searchQuery ? 'No matching children going home early' : 'No children going home early' }}
          </div>
        </div>
      </div>

      <!-- Course Sections -->
      <div v-for="course in filteredCourses" :key="course.id" class="course-distribution__course" :class="[
        getCourseStatus(course) === 'overfilled' ? 'course-distribution__course--overfilled' : '',
        getCourseStatus(course) === 'near_full' ? 'course-distribution__course--near-full' : '',
        getCourseStatus(course) === 'locked' ? 'course-distribution__course--locked' : ''
      ]">
        <div class="course-distribution__course-header">
          <h4 class="course-distribution__course-title">
            <button 
              class="course-distribution__course-name" 
              @click="toggleCourseDetails(course.id)" 
              :title="course.name"
            >
              {{ course.name }}
            </button>

            <!-- Capacity badge with status colors -->
            <span class="course-distribution__course-capacity" :class="capacityPillClass(course)">
              {{ displayRosterCount(course) }}/{{ course.maxCapacity || course.max_capacity }}
            </span>

            <!-- Overfilled pulsing badge -->
            <span v-if="getCourseStatus(course) === 'overfilled'" class="course-distribution__course-overfilled-badge" aria-live="polite">
              OVERFILLED - NEEDS REDISTRIBUTION
            </span>

            <!-- Search results indicator -->
            <span v-if="searchQuery && (enrolledByCourseId[course.id] || []).length > 0" class="course-distribution__course-search-badge">
              {{ (enrolledByCourseId[course.id] || []).length }} shown
            </span>
          </h4>

          <div class="course-distribution__course-actions">
            <!-- Lock/Unlock Button -->
            <button
              v-if="course.is_locked"
              class="course-distribution__course-lock-btn course-distribution__course-lock-btn--locked"
              :title="'Click to open course'"
              @click="onLockToggle(course.id, false)"
            >
              <svg class="course-distribution__course-lock-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 15v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2zm10-10V7a4 4 0 00-8 0v4h8z" />
              </svg>
              Closed
            </button>
            <button
              v-else
              class="course-distribution__course-lock-btn"
              :title="'Mark as closed'"
              @click="onLockToggle(course.id, true)"
            >
              <svg class="course-distribution__course-lock-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 11V7a4 4 0 118 0m-4 8v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2z" />
              </svg>
            </button>

            <!-- Approval Button -->
            <button 
              class="course-distribution__course-approval-btn"
              :class="{ 'course-distribution__course-approval-btn--approved': isCourseApproved(course.id) }"
              :title="isCourseApproved(course.id) ? 'Mark as needs review' : 'Mark as approved'"
              @click="onApproveClick(course.id)"
            >
              <svg class="course-distribution__course-approval-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
              </svg>
              <span v-if="isCourseApproved(course.id)" class="course-distribution__course-approval-text">Approved</span>
            </button>

            <!-- Notes Button -->
            <button 
              class="course-distribution__course-notes-btn" 
              title="Add note" 
              @click="onNotesClick(course.id)"
            >
              <svg class="course-distribution__course-notes-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z" />
              </svg>
              <span v-if="course.notes_count" class="course-distribution__course-notes-badge">{{ course.notes_count }}</span>
            </button>
          </div>
        </div>

        <!-- Course Details Section - Only show when expanded -->
        <div v-if="isCourseDetailsVisible(course.id)" class="course-distribution__course-details">
          <div class="course-distribution__course-meta">
            <span v-if="course.teacher" class="course-distribution__course-meta-item">Teacher: {{ course.teacher }}</span>
            <span v-if="course.room" class="course-distribution__course-meta-item">Room: {{ course.room }}</span>
            <span v-if="course.grades" class="course-distribution__course-meta-item">Grades: {{ course.grades.join(', ') }}</span>
            <div v-if="course.notes_count > 0" class="course-distribution__course-notes-indicator">
              <span class="course-distribution__course-notes-badge-small">
                <svg class="course-distribution__course-notes-icon-small" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z" />
                </svg>
                {{ course.notes_count }} note{{ course.notes_count > 1 ? 's' : '' }}
              </span>
              <button
                class="course-distribution__course-notes-toggle"
                @click="toggleNotesVisibility(course.id)"
                :title="isNotesVisible(course.id) ? 'Hide notes' : 'Show notes'"
              >
                <svg v-if="isNotesVisible(course.id)" class="course-distribution__course-notes-toggle-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13.875 18.825A10.05 10.05 0 0112 19c-4.478 0-8.268-2.943-9.543-7a9.97 9.97 0 011.563-3.029m5.858.908a3 3 0 114.243 4.243M9.878 9.878l4.242 4.242M9.878 9.878L3 3m6.878 6.878L21 21" />
                </svg>
                <svg v-else class="course-distribution__course-notes-toggle-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z" />
                </svg>
              </button>
            </div>
          </div>
        </div>

        <!-- Participants Section -->
        <div class="course-distribution__course-participants">
          <!-- Header row aligned with columns + chevron -->
                     <div class="course-distribution__course-participants-header">
             <div class="course-distribution__course-participants-grid">
               <div class="course-distribution__course-participants-col course-distribution__course-participants-col--participants">Participants</div>
               <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wish">1. wish</div>
               <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wish">2. wish</div>
               <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wish">3. wish</div>
               <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wait">Wait</div>
             </div>
             <div class="course-distribution__course-participants-toggle-container">
               <button
                 class="course-distribution__course-participants-toggle"
                 :title="isParticipantsVisible(course.id) ? 'Hide participants' : 'Show participants'"
                 @click="toggleParticipants(course.id)"
               >
                 <svg v-if="isParticipantsVisible(course.id)" class="course-distribution__course-participants-toggle-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                   <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 15l7-7 7 7" />
                 </svg>
                 <svg v-else class="course-distribution__course-participants-toggle-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                   <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                 </svg>
               </button>
             </div>
           </div>

                      <template v-if="isParticipantsVisible(course.id)">
              <div v-for="student in (enrolledByCourseId[course.id] || [])" :key="student.id + '-' + selectedDayNumber"
               class="course-distribution__course-participants-row">
               <div class="course-distribution__course-participants-grid">
                 <div class="course-distribution__course-participants-col course-distribution__course-participants-col--participants">
                   <span class="course-distribution__course-participants-student-name">{{ student.class }} {{ student.name }}</span>
                   <div class="course-distribution__course-participants-student-schedule">
                     <span 
                       v-for="(enrollment, dayIndex) in student.enrollments" 
                       :key="dayIndex"
                       class="course-distribution__course-participants-schedule-chip"
                     >
                       {{ getDayName(dayIndex) }} ({{ parseInt(dayIndex) + 1 }}): {{ enrollment || 'Home' }}
                     </span>
                   </div>
                 </div>

                 <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wish">
                   <button
                     v-if="student.first_choice"
                     class="course-distribution__course-participants-preference-btn"
                     :class="[
                       isCurrentChoice(student, 1) ? 'course-distribution__course-participants-preference-btn--first-choice' : '',
                       student.first_choice === 'go-home' ? 'course-distribution__course-participants-preference-btn--go-home' : '',
                       isLockedTarget(student.first_choice) ? 'course-distribution__course-participants-preference-btn--locked' : ''
                     ]"
                     :disabled="isCurrentChoice(student, 1) || isLockedTarget(student.first_choice)"
                     @click="!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice) && onStudentMove(student.id, student.first_choice)"
                     :title="isLockedTarget(student.first_choice) ? 'Course locked' : formatPreferenceDisplay(student.first_choice)"
                   >
                     {{ formatPreferenceDisplay(student.first_choice) }}
                   </button>
                 </div>

                 <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wish">
                   <button
                     v-if="student.second_choice"
                     class="course-distribution__course-participants-preference-btn"
                     :class="[
                       isCurrentChoice(student, 2) ? 'course-distribution__course-participants-preference-btn--second-choice' : '',
                       student.second_choice === 'go-home' ? 'course-distribution__course-participants-preference-btn--go-home' : '',
                       isLockedTarget(student.second_choice) ? 'course-distribution__course-participants-preference-btn--locked' : ''
                     ]"
                     :disabled="isCurrentChoice(student, 2) || isLockedTarget(student.second_choice)"
                     @click="!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice) && onStudentMove(student.id, student.second_choice)"
                     :title="isLockedTarget(student.second_choice) ? 'Course locked' : formatPreferenceDisplay(student.second_choice)"
                   >
                     {{ formatPreferenceDisplay(student.second_choice) }}
                   </button>
                 </div>

                 <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wish">
                   <button
                     v-if="student.third_choice"
                     class="course-distribution__course-participants-preference-btn"
                     :class="[
                       isCurrentChoice(student, 3) ? 'course-distribution__course-participants-preference-btn--third-choice' : '',
                       student.third_choice === 'go-home' ? 'course-distribution__course-participants-preference-btn--go-home' : '',
                       isLockedTarget(student.third_choice) ? 'course-distribution__course-participants-preference-btn--locked' : ''
                     ]"
                     :disabled="isCurrentChoice(student, 3) || isLockedTarget(student.third_choice)"
                     @click="!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice) && onStudentMove(student.id, student.third_choice)"
                     :title="isLockedTarget(student.third_choice) ? 'Course locked' : formatPreferenceDisplay(student.third_choice)"
                   >
                     {{ formatPreferenceDisplay(student.third_choice) }}
                   </button>
                 </div>

                 <div class="course-distribution__course-participants-col course-distribution__course-participants-col--wait">
                   <button
                     v-if="!hasGoHomePreference(student)"
                     class="course-distribution__course-participants-wait-btn"
                     title="Move to waiting list"
                     @click="onStudentMove(student.id, 'waiting')"
                   >
                     W
                   </button>
                   <div v-else class="course-distribution__course-participants-wait-placeholder"></div>
                 </div>
               </div>
               <div class="course-distribution__course-participants-spacer">
                 <!-- Empty space to match the toggle button area in the header -->
               </div>
             </div>

            <p v-if="(enrolledByCourseId[course.id] || []).length === 0" class="course-distribution__course-participants-empty">No students enrolled</p>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { computed, ref, onMounted } from 'vue';

export default {
  props: {
    content: { type: Object, required: true },
    uid: { type: String, required: true },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  },
  emits: ['trigger-event'],
  setup(props, { emit }) {
    const isEditing = computed(() => {
      /* wwEditor:start */ return props.wwEditorState.isEditing; /* wwEditor:end */
      return false;
    });

    // State
    const loading = ref(true);
    const searchQuery = ref('');
    const selectedDayNumber = ref(1);
    const selectedSemesterIndex = ref(0);
    const allStudents = ref([]);
    const allCourses = ref([]);
    const daysOfWeek = ref([]);
    const semesters = ref([]);
    const currentSemester = ref('');
    const courseNotes = ref([]);
    const visibleParticipants = ref(new Set());
    const collapsedCourseDetails = ref(new Set());
    const isGoingHomeExpanded = ref(true);
    const isGoingHomeChildrenVisible = ref(true);
    const notesVisible = ref(new Set());
    const approvedCourses = ref(new Set());


    // Mock data for WeWeb compatibility
    const initializeMockData = () => {
      daysOfWeek.value = [
        { day_id: 1, day_number: 1, name_en: 'Monday' },
        { day_id: 2, day_number: 2, name_en: 'Tuesday' },
        { day_id: 3, day_number: 3, name_en: 'Wednesday' },
        { day_id: 4, day_number: 4, name_en: 'Thursday' },
        { day_id: 5, day_number: 5, name_en: 'Friday' }
      ];

      semesters.value = [
        {
          id: 'fall2024',
          name: 'Fall 2024',
          schedule: [
            {
              day: 'Monday',
              courses: [
                { id: 'course-1', name: 'Computer', maxCapacity: 12, enrolledChildren: ['student-2', 'student-3'], teacher: 'Steffi J', room: 'Computer', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-2', name: 'Lego-Bau', maxCapacity: 10, enrolledChildren: [], teacher: 'Anna K', room: 'Werkraum', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-3', name: 'Lese-Club', maxCapacity: 8, enrolledChildren: [], teacher: 'Maria L', room: 'Bibliothek', availableGrades: ['3', '4', '5'], notes: [{ id: 'note1', text: 'Need to check equipment before class starts', author: 'Admin', timestamp: new Date(2024, 0, 15, 10, 30), isProblem: true, isResolved: false }], forcedFull: false },
              ]
            },
            {
              day: 'Tuesday',
              courses: [
                { id: 'course-1', name: 'Computer', maxCapacity: 12, enrolledChildren: ['student-1', 'student-2'], teacher: 'Steffi J', room: 'Computer', availableGrades: ['3', '4', '5'], notes: [{ id: 'note2', text: 'Computer equipment needs maintenance', author: 'Steffi J', timestamp: new Date(2024, 0, 16, 14, 20), isProblem: true, isResolved: false }], forcedFull: false },
                { id: 'course-2', name: 'Lego-Bau', maxCapacity: 10, enrolledChildren: ['student-3'], teacher: 'Anna K', room: 'Werkraum', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-3', name: 'Lese-Club', maxCapacity: 8, enrolledChildren: [], teacher: 'Maria L', room: 'Bibliothek', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
              ]
            },
            {
              day: 'Wednesday',
              courses: [
                { id: 'course-1', name: 'Computer', maxCapacity: 12, enrolledChildren: ['student-1'], teacher: 'Steffi J', room: 'Computer', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-2', name: 'Lego-Bau', maxCapacity: 10, enrolledChildren: ['student-2', 'student-3'], teacher: 'Anna K', room: 'Werkraum', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-3', name: 'Lese-Club', maxCapacity: 8, enrolledChildren: [], teacher: 'Maria L', room: 'Bibliothek', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
              ]
            }
          ],
          children: [
            { id: 'student-1', name: 'Anna Meier', class: '3A', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'course-3', enrollments: { 0: undefined, 1: 'course-1', 2: 'course-1' } },
            { id: 'student-2', name: 'Max Schmidt', class: '4B', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'course-3', enrollments: { 0: 'course-1', 1: 'course-1', 2: 'course-2' } },
            { id: 'student-3', name: 'Lea Müller', class: '5A', first_choice: 'course-1', second_choice: 'course-3', third_choice: 'course-2', enrollments: { 0: 'course-1', 1: 'course-2', 2: 'course-2' } },
            { id: 'student-4', name: 'Tom Weber', class: '3B', first_choice: 'course-2', second_choice: 'course-1', third_choice: 'course-3', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-5', name: 'Lisa Fischer', class: '4A', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'course-2', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-6', name: 'Paul Wagner', class: '5B', first_choice: 'course-2', second_choice: 'course-3', third_choice: 'course-1', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-7', name: 'Emma Schulz', class: '3A', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'go-home', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-8', name: 'Felix Klein', class: '4B', first_choice: 'course-2', second_choice: 'course-1', third_choice: 'go-home', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-9', name: 'Nina Hoffmann', class: '5A', first_choice: 'course-3', second_choice: 'course-2', third_choice: 'course-1', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-10', name: 'Lukas Meyer', class: '3B', first_choice: 'course-1', second_choice: 'course-3', third_choice: 'course-2', enrollments: { 0: undefined, 1: undefined, 2: undefined } }
          ]
        },
        {
          id: 'spring2025',
          name: 'Spring 2025',
          schedule: [
            {
              day: 'Monday',
              courses: [
                { id: 'course-1', name: 'Computer', maxCapacity: 12, enrolledChildren: [], teacher: 'Steffi J', room: 'Computer', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-2', name: 'Lego-Bau', maxCapacity: 10, enrolledChildren: [], teacher: 'Anna K', room: 'Werkraum', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-3', name: 'Lese-Club', maxCapacity: 8, enrolledChildren: [], teacher: 'Maria L', room: 'Bibliothek', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
              ]
            },
            {
              day: 'Tuesday',
              courses: [
                { id: 'course-1', name: 'Computer', maxCapacity: 12, enrolledChildren: [], teacher: 'Steffi J', room: 'Computer', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-2', name: 'Lego-Bau', maxCapacity: 10, enrolledChildren: [], teacher: 'Anna K', room: 'Werkraum', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-3', name: 'Lese-Club', maxCapacity: 8, enrolledChildren: [], teacher: 'Maria L', room: 'Bibliothek', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
              ]
            },
            {
              day: 'Wednesday',
              courses: [
                { id: 'course-1', name: 'Computer', maxCapacity: 12, enrolledChildren: [], teacher: 'Steffi J', room: 'Computer', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-2', name: 'Lego-Bau', maxCapacity: 10, enrolledChildren: [], teacher: 'Anna K', room: 'Werkraum', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
                { id: 'course-3', name: 'Lese-Club', maxCapacity: 8, enrolledChildren: [], teacher: 'Maria L', room: 'Bibliothek', availableGrades: ['3', '4', '5'], notes: [], forcedFull: false },
              ]
            }
          ],
          children: [
            { id: 'student-1', name: 'Anna Meier', class: '3A', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'course-3', enrollments: {} },
            { id: 'student-2', name: 'Max Schmidt', class: '4B', first_choice: 'course-2', second_choice: 'course-1', third_choice: 'course-3', enrollments: {} },
            { id: 'student-3', name: 'Lea Müller', class: '5A', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'course-2', enrollments: {} },
            { id: 'student-4', name: 'Tom Weber', class: '3B', first_choice: 'course-1', second_choice: 'course-3', third_choice: 'course-2', enrollments: {} },
            { id: 'student-5', name: 'Lisa Fischer', class: '4A', first_choice: 'course-2', second_choice: 'course-1', third_choice: 'course-3', enrollments: {} },
            { id: 'student-6', name: 'Paul Wagner', class: '5B', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'course-3', enrollments: {} },
            { id: 'student-7', name: 'Emma Schulz', class: '3A', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'go-home', enrollments: {} },
            { id: 'student-8', name: 'Felix Klein', class: '4B', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'go-home', enrollments: {} },
            { id: 'student-9', name: 'Nina Hoffmann', class: '5A', first_choice: 'course-2', second_choice: 'course-3', third_choice: 'course-1', enrollments: {} },
            { id: 'student-10', name: 'Lukas Meyer', class: '3B', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'course-2', enrollments: {} }
          ]
        }
      ];

      // Sync student enrollments with schedule.enrolledChildren for each day
      semesters.value.forEach((semester) => {
        const childrenById = Object.fromEntries(
          (semester.children || []).map((c) => [c.id, c])
        );
        (semester.schedule || []).forEach((dayEntry, dayIdx) => {
          (dayEntry.courses || []).forEach((course) => {
            (course.enrolledChildren || []).forEach((studentId) => {
              const student = childrenById[studentId];
              if (student) {
                if (!student.enrollments) student.enrollments = {};
                student.enrollments[dayIdx] = course.id;
              }
            });
          });
        });
      });

      allCourses.value = [
        {
          id: 'course-1',
          name: 'Computer',
          teacher: 'Steffi J',
          room: 'Computer',
          day: 'Monday',
          maxCapacity: 12,
          is_locked: false,
          notes_count: 0,
          forcedFull: false
        },
        {
          id: 'course-2', 
          name: 'Lego-Bau',
          teacher: 'Anna K',
          room: 'Werkraum',
          day: 'Monday',
          max_capacity: 10,
          is_locked: false,
          notes_count: 0
        },
        {
          id: 'course-3',
          name: 'Lese-Club',
          teacher: 'Maria L',
          room: 'Bibliothek',
          day: 'Monday', 
          max_capacity: 8,
          is_locked: false,
          notes_count: 0
        }
      ];

      allStudents.value = [
        {
          id: 'student-1',
          name: 'Anna Meier',
          current_enrollment: null,
          first_choice: 'course-1',
          second_choice: 'course-2',
          third_choice: 'course-3'
        },
        {
          id: 'student-2',
          name: 'Max Schmidt', 
          current_enrollment: 'course-1',
          first_choice: 'course-1',
          second_choice: 'course-2',
          third_choice: 'course-3'
        },
        {
          id: 'student-3',
          name: 'Lea Müller',
          current_enrollment: 'course-1',
          first_choice: 'course-1',
          second_choice: 'course-3',
          third_choice: 'course-2'
        }
      ];

      courseNotes.value = [
        {
          id: 'note-1',
          course_id: 'course-1',
          text: 'Need to check equipment before class starts',
          author: 'Test Admin',
          is_problem: true,
          is_resolved: false,
          created_at: new Date().toISOString()
        }
      ];

      currentSemester.value = semesters.value[0].name;
      loading.value = false;
    };

    // Computed properties
    const currentSemesterData = computed(() => {
      return semesters.value[selectedSemesterIndex.value] || semesters.value[0];
    });

    const currentDayData = computed(() => {
      const semester = currentSemesterData.value;
      return semester?.schedule[selectedDayNumber.value - 1] || semester?.schedule[0];
    });

    const selectedDayName = computed(() => {
      const day = currentDayData.value;
      return day?.day || 'Monday';
    });

    const filteredStudents = computed(() => {
      const students = currentSemesterData.value?.children || [];
      if (!searchQuery.value.trim()) return students;
      const query = searchQuery.value.toLowerCase();
      return students.filter(student => 
        student.name.toLowerCase().includes(query)
      );
    });

    const filteredCourses = computed(() => {
      return currentDayData.value?.courses || [];
    });

    const waitingListStudents = computed(() => {
      return filteredStudents.value.filter(student => 
        !student.enrollments || student.enrollments[selectedDayNumber.value - 1] === undefined
      );
    });

    const goingHomeStudents = computed(() => {
      return filteredStudents.value.filter(student => 
        student.enrollments && student.enrollments[selectedDayNumber.value - 1] === undefined &&
        (student.first_choice === 'go-home' || student.second_choice === 'go-home' || student.third_choice === 'go-home')
      );
    });

    const firstChoiceGoHomeStudents = computed(() => {
      return filteredStudents.value.filter(student => 
        student.enrollments && student.enrollments[selectedDayNumber.value - 1] === undefined &&
        student.first_choice === 'go-home'
      );
    });

    const secondOrThirdChoiceGoHomeStudents = computed(() => {
      return filteredStudents.value.filter(student => 
        student.enrollments && student.enrollments[selectedDayNumber.value - 1] === undefined &&
        student.first_choice !== 'go-home' && (student.second_choice === 'go-home' || student.third_choice === 'go-home')
      );
    });

    const firstChoiceGoHomeClassStats = computed(() => {
      const firstChoiceChildren = firstChoiceGoHomeStudents.value;
      const allChildren = currentSemesterData.value?.children || [];
      
      // Group all children by class
      const classCounts = {};
      
      allChildren.forEach(child => {
        const className = child.class;
        if (!classCounts[className]) {
          classCounts[className] = { total: 0, goingHome: 0 };
        }
        classCounts[className].total++;
      });
      
      // Count going home children by class
      firstChoiceChildren.forEach(child => {
        const className = child.class;
        if (classCounts[className]) {
          classCounts[className].goingHome++;
        }
      });
      
      // Convert to array and sort by class name
      return Object.entries(classCounts)
        .filter(([_, stats]) => stats.goingHome > 0)
        .map(([className, stats]) => ({
          className,
          goingHome: stats.goingHome,
          total: stats.total,
          children: firstChoiceChildren.filter(child => child.class === className)
        }))
        .sort((a, b) => a.className.localeCompare(b.className));
    });

    const totalStudentsInSchool = computed(() => {
      return currentSemesterData.value?.children?.length || 0;
    });



    const enrolledByCourseId = computed(() => {
      const enrolled = {};
      const courses = filteredCourses.value;
      const students = filteredStudents.value;
      
      courses.forEach(course => {
        enrolled[course.id] = students.filter(student => 
          student.enrollments && student.enrollments[selectedDayNumber.value - 1] === course.id
        );
      });
      return enrolled;
    });

    const totalEnrolledForDay = computed(() => {
      return filteredStudents.value.filter(student => student.current_enrollment).length;
    });

    const unresolvedProblems = computed(() => {
      return courseNotes.value.filter(note => note.is_problem && !note.is_resolved);
    });

    // Helper functions
    const getCourseName = (courseId) => {
      const course = allCourses.value.find(c => c.id === courseId);
      return course?.name || 'Unknown Course';
    };

    const formatDate = (dateString) => {
      if (!dateString) return '';
      const date = new Date(dateString);
      return new Intl.DateTimeFormat('en-US', {
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      }).format(date);
    };

    const formatPreferenceDisplay = (courseId) => {
      if (courseId === 'go-home') return 'Home';
      const course = filteredCourses.value.find(c => c.id === courseId);
      return course?.name || 'Unknown';
    };

    const isCurrentChoice = (student, choiceNumber) => {
      const choice = choiceNumber === 1 ? student.first_choice :
                   choiceNumber === 2 ? student.second_choice : 
                   student.third_choice;
      return student.enrollments && student.enrollments[selectedDayNumber.value - 1] === choice;
    };

    const isLockedTarget = (courseId) => {
      if (!courseId || courseId === 'go-home') return false;
      const course = allCourses.value.find(c => c.id === courseId);
      return !!course?.is_locked;
    };

    const displayRosterCount = (course) => {
      return enrolledByCourseId.value[course.id]?.length || 0;
    };

    const getCourseStatus = (course) => {
      const enrolled = displayRosterCount(course);
      const capacity = course.maxCapacity || course.max_capacity;
      
      if (course.is_locked || course.forcedFull) return 'locked';
      if (enrolled > capacity) return 'overfilled';
      if (enrolled >= capacity * 0.8) return 'near_full';
      return 'normal';
    };

    const statusSectionClass = (course) => {
      const status = getCourseStatus(course);
      return {
        'course-distribution__section--near-full': status === 'near_full',
        'course-distribution__section--overfilled': status === 'overfilled',
        'course-distribution__section--locked': status === 'locked'
      };
    };

    const capacityPillClass = (course) => {
      const status = getCourseStatus(course);
      return {
        'cd-capacity--normal': status === 'normal',
        'cd-capacity--near': status === 'near_full',
        'cd-capacity--over': status === 'overfilled',
        'cd-capacity--locked': status === 'locked'
      };
    };

    const statusBadgeText = (course) => {
      const status = getCourseStatus(course);
      return status === 'overfilled' ? 'Overfilled' : '';
    };

    const isParticipantsVisible = (courseId) => {
      // Default to visible when not explicitly toggled
      return !visibleParticipants.value.has(courseId);
    };

    const toggleParticipants = (courseId) => {
      if (visibleParticipants.value.has(courseId)) {
        visibleParticipants.value.delete(courseId);
      } else {
        visibleParticipants.value.add(courseId);
      }
    };

    const toggleCourseDetails = (courseId) => {
      if (collapsedCourseDetails.value.has(courseId)) {
        collapsedCourseDetails.value.delete(courseId);
      } else {
        collapsedCourseDetails.value.add(courseId);
      }
    };

    const isCourseDetailsVisible = (courseId) => {
      return !collapsedCourseDetails.value.has(courseId);
    };

    const isNotesVisible = (courseId) => {
      return notesVisible.value.has(courseId);
    };

    const toggleNotesVisibility = (courseId) => {
      if (notesVisible.value.has(courseId)) {
        notesVisible.value.delete(courseId);
      } else {
        notesVisible.value.add(courseId);
      }
    };

    const isCourseApproved = (courseId) => {
      return approvedCourses.value.has(courseId);
    };

    const hasGoHomePreference = (student) => {
      return student.first_choice === 'go-home' || 
             student.second_choice === 'go-home' || 
             student.third_choice === 'go-home';
    };

    const getDayName = (dayIndex) => {
      const days = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri'];
      return days[dayIndex] || 'Unknown';
    };



    // Event handlers
    const onDayChange = async (day) => {
      if (isEditing.value) return;
      selectedDayNumber.value = parseInt(day, 10);
      emit('trigger-event', { name: 'dayChange', event: { value: selectedDayNumber.value } });
    };

    const onSearchChange = (query) => {
      if (isEditing.value) return;
      searchQuery.value = query;
      emit('trigger-event', { name: 'searchChange', event: { value: query } });
    };

    const onStudentMove = async (studentId, targetId) => {
      if (isEditing.value) return;
      console.log('Moving student', studentId, 'to', targetId);
      
      // Update student enrollment in current semester data
      const semester = currentSemesterData.value;
      const student = semester.children.find(s => s.id === studentId);
      if (student) {
        if (targetId === 'waiting') {
          student.enrollments[selectedDayNumber.value - 1] = undefined;
        } else {
          student.enrollments[selectedDayNumber.value - 1] = targetId;
        }
      }
      
      emit('trigger-event', { name: 'studentMove', event: { studentId, courseId: targetId } });
    };

    const onLockToggle = async (courseId, lockState) => {
      if (isEditing.value) return;
      console.log('Toggling lock for course', courseId, 'to', lockState);
      
      // Update course lock state in mock data
      const course = allCourses.value.find(c => c.id === courseId);
      if (course) {
        course.is_locked = lockState;
      }
      // Also reflect lock state on the course objects used in the current view
      const day = currentDayData.value;
      if (day && Array.isArray(day.courses)) {
        const target = day.courses.find(c => c.id === courseId);
        if (target) {
          target.is_locked = lockState;
        }
      }
      
      emit('trigger-event', { name: 'courseLockToggle', event: { courseId, lockState } });
    };

    const onApproveClick = (courseId) => {
      if (isEditing.value) return;
      console.log('Approving course', courseId);
      if (approvedCourses.value.has(courseId)) {
        approvedCourses.value.delete(courseId);
      } else {
        approvedCourses.value.add(courseId);
      }
      emit('trigger-event', { name: 'courseApprove', event: { courseId } });
    };

    const onNotesClick = (courseId) => {
      if (isEditing.value) return;
      console.log('Opening notes for course', courseId);
      emit('trigger-event', { name: 'courseNotesOpen', event: { courseId } });
    };

    const onResolveNote = async (noteId) => {
      if (isEditing.value) return;
      console.log('Resolving note', noteId);
      
      // Update note in mock data
      const note = courseNotes.value.find(n => n.id === noteId);
      if (note) {
        note.is_resolved = true;
        note.resolved_at = new Date().toISOString();
      }
      
      emit('trigger-event', { name: 'problemResolved', event: { noteId } });
    };

    const onResetToFirstChoices = () => {
      if (isEditing.value) return;
      console.log('Resetting all students to first choices');
      
      // Reset all students to their first choice for current day
      const semester = currentSemesterData.value;
      semester.children.forEach(student => {
        if (student.first_choice && student.first_choice !== 'go-home') {
          student.enrollments[selectedDayNumber.value - 1] = student.first_choice;
        } else {
          student.enrollments[selectedDayNumber.value - 1] = undefined;
        }
      });
      
      emit('trigger-event', { name: 'resetToFirstChoices' });
    };

    const onSemesterChange = (index) => {
      if (isEditing.value) return;
      selectedSemesterIndex.value = parseInt(index);
      currentSemester.value = semesters.value[selectedSemesterIndex.value].name;
      emit('trigger-event', { name: 'semesterChange', event: { value: selectedSemesterIndex.value } });
    };

    // Initialize data when component mounts
    onMounted(() => {
      initializeMockData();
    });

    return {
      // State
      loading,
      searchQuery,
      selectedDayNumber,
      selectedSemesterIndex,
      currentSemester,
      courseNotes,
      isGoingHomeExpanded,
      isGoingHomeChildrenVisible,
      
      // Computed
      selectedDayName,
      filteredCourses,
      waitingListStudents,
      goingHomeStudents,
      firstChoiceGoHomeStudents,
      secondOrThirdChoiceGoHomeStudents,
      firstChoiceGoHomeClassStats,
      totalStudentsInSchool,
      enrolledByCourseId,
      totalEnrolledForDay,
      unresolvedProblems,
      daysOfWeek,
      semesters,
      allCourses,
      
      // Helper functions
      getCourseName,
      formatDate,
      formatPreferenceDisplay,
      isCurrentChoice,
      isLockedTarget,
      displayRosterCount,
      getCourseStatus,
      statusSectionClass,
      capacityPillClass,
      statusBadgeText,
      isCourseDetailsVisible,
      isNotesVisible,
      toggleNotesVisibility,
      isCourseApproved,
      isParticipantsVisible,
      toggleParticipants,
      toggleCourseDetails,
      hasGoHomePreference,
      getDayName,
      
      // Event handlers
      onDayChange,
      onSearchChange,
      onStudentMove,
      onLockToggle,
      onApproveClick,
      onNotesClick,
      onResolveNote,
      onResetToFirstChoices,
      onSemesterChange,
      
      // Props
      content: props.content,
    };
  },
};
</script>

<style lang="scss" scoped>
/* Container */
.course-distribution {
  display: flex;
  flex-direction: column;
  width: 100%;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  background-color: #ffffff;

  &__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px 24px;
    border-bottom: 1px solid #e5e7eb;
    background-color: #ffffff;
  }

  &__header-left {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__header-right {
    display: flex;
    gap: 16px;
    align-items: center;
  }

  &__reset-btn {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 8px 12px;
    background-color: #f3f4f6;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 14px;
    font-weight: 500;
    color: #374151;
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
      border-color: #9ca3af;
    }

    &:active {
      background-color: #d1d5db;
    }
  }

  &__reset-icon {
    width: 16px;
    height: 16px;
    stroke: currentColor;
    fill: none;
  }

  &__semester-selector {
    display: flex;
    align-items: center;
  }

  &__semester-select {
    padding: 8px 12px;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 14px;
    background-color: #ffffff;
    min-width: 120px;

    &:focus {
      outline: none;
      border-color: #3b82f6;
      box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
  }

  &__going-home {
    margin-bottom: 16px;
    padding: 12px;
    background-color: rgba(243, 244, 246, 0.3);
    border-radius: 8px;
    border-left: 2px solid #9ca3af;
  }

  &__going-home-header {
    display: flex;
    align-items: center;
    gap: 8px;
    cursor: pointer;
    padding: 4px;
    margin: -4px;
    border-radius: 4px;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: rgba(229, 231, 235, 0.5);
    }
  }

  &__going-home-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__going-home-title {
    font-size: 14px;
    font-weight: 400;
    margin: 0;
    color: #374151;
  }

  &__going-home-badge {
    font-size: 12px;
    background-color: #e5e7eb;
    color: #374151;
    padding: 4px 8px;
    border-radius: 4px;
    font-weight: 400;
  }

  &__going-home-arrow {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
    transition: transform 0.2s ease;

    &--expanded {
      transform: rotate(90deg);
    }
  }

  &__going-home-content {
    margin-top: 12px;
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  &__going-home-section {
    display: flex;
    flex-direction: column;
  }

  &__going-home-section-title {
    font-size: 14px;
    font-weight: 500;
    margin-bottom: 8px;
    color: #6b7280;
  }

  &__going-home-classes {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__going-home-class {
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    padding: 8px;
    background-color: rgba(255, 255, 255, 0.5);
  }

  &__going-home-class-header {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 8px;
  }

  &__going-home-class-name {
    font-size: 14px;
    font-weight: 500;
    color: #374151;
  }

  &__going-home-class-badge {
    font-size: 12px;
    background-color: #f3f4f6;
    color: #374151;
    border: 1px solid #d1d5db;
    padding: 4px 8px;
    border-radius: 4px;
    font-weight: 400;
  }

  &__going-home-students {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-left: 8px;
  }

  &__going-home-student {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 4px 8px;
    background-color: rgba(255, 255, 255, 0.5);
    border-radius: 4px;
    font-size: 14px;
  }

  &__going-home-student-name {
    color: #374151;
  }

  &__going-home-student-schedule {
    display: flex;
    gap: 4px;
    font-size: 12px;
    color: #6b7280;
  }

  &__going-home-schedule-chip {
    cursor: pointer;
    padding: 2px 4px;
    border-radius: 2px;
    font-weight: 500;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
    }
  }

  &__going-home-secondary {
    border-radius: 8px;
    padding: 12px;
    background-color: #fef3c7;
    border: 1px solid #fbbf24;
  }

  &__going-home-secondary-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 8px;
  }

  &__going-home-secondary-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 14px;
    font-weight: 500;
    color: #374151;
  }

  &__going-home-secondary-badge {
    font-size: 12px;
    background-color: #fed7aa;
    color: #c2410c;
    border: 1px solid #fb923c;
    padding: 4px 8px;
    border-radius: 4px;
    font-weight: 400;
  }

  &__going-home-grid {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__going-home-grid-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 4px 8px;
    background-color: rgba(243, 244, 246, 0.5);
    border-radius: 4px;
    font-size: 12px;
    font-weight: 500;
    color: #6b7280;
    border-bottom: 1px solid #e5e7eb;
  }

  &__going-home-grid-columns {
    display: grid;
    grid-template-columns: 5fr 2fr 2fr 2fr 1fr;
    gap: 8px;
    flex: 1;
  }

  &__going-home-grid-col {
    display: flex;
    justify-content: center;
    align-items: center;
    text-align: center;
    font-size: 12px;
    font-weight: 500;

    &--participants {
      justify-content: flex-start;
      text-align: left;
    }

    &--wish {
      justify-content: center;
    }

    &--wait {
      justify-content: center;
    }
  }

  &__going-home-toggle {
    margin-left: 8px;
  }

  &__going-home-toggle-btn {
    width: 16px;
    height: 16px;
    padding: 0;
    background-color: transparent;
    border: 0;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__going-home-toggle-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;

    &--up {
      color: #2563eb;
    }

    &--down {
      color: #6b7280;
    }
  }

  &__going-home-grid-rows {
    display: flex;
    flex-direction: column;
  }

  &__going-home-grid-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 4px 8px;
    background-color: #ffffff;
    border-radius: 4px;
    font-size: 14px;
  }

  &__going-home-student-info {
    color: #374151;
    font-weight: 400;
  }

  &__going-home-preference-btn {
    height: 24px;
    padding: 0 8px;
    font-size: 12px;
    width: 100%;
    background-color: transparent;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    transition: all 0.2s ease;
    cursor: pointer;

    &:hover {
      background-color: #f9fafb;
    }

    &:disabled {
      cursor: default;
    }

    &--current {
      background-color: #dbeafe;
      border-color: #93c5fd;
    }

    &--go-home {
      border-color: #f59e0b;
      background-color: #fef3c7;
    }

    &--third {
      border: 2px solid #f59e0b;
      background-color: #fef3c7;

      &:hover {
        background-color: #fde68a;
      }
    }

    // React-style preference border classes
    &--first-choice {
      border: 2px solid #22c55e;
      background-color: #f0fdf4;
    }

    &--second-choice {
      border: 2px solid #3b82f6;
      background-color: #eff6ff;
    }

    &--third-choice {
      border: 2px solid #f97316;
      background-color: #fff7ed;
    }
  }

  &__going-home-wait-btn {
    height: 24px;
    width: 24px;
    padding: 0;
    font-size: 12px;
    background-color: #f3f4f6;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
    }
  }

  &__going-home-wait-placeholder {
    height: 24px;
    width: 24px;
  }

  &__going-home-grid-spacer {
    margin-left: 8px;
    width: 16px;
  }

  &__going-home-toggle-hidden {
    display: flex;
    justify-content: flex-end;
    margin-top: 8px;
  }

  &__going-home-empty {
    text-align: center;
    padding: 8px 0;
    font-size: 12px;
    color: #6b7280;
  }

  &__waiting-list {
    margin-bottom: 16px;
    padding: 12px;
    background-color: rgba(243, 244, 246, 0.3);
    border-radius: 8px;
    border-left: 2px solid #6b7280;
  }

  &__waiting-list-header {
    margin-bottom: 8px;
  }

  &__waiting-list-title {
    font-size: 14px;
    font-weight: 500;
    color: #374151;
    margin: 0;
    display: flex;
    align-items: center;
    gap: 4px;
  }

  &__waiting-list-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__waiting-list-content {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__waiting-list-header-row {
    display: grid;
    grid-template-columns: 5fr 2fr 2fr 2fr 1fr;
    gap: 8px;
    padding: 4px 8px;
    background-color: rgba(243, 244, 246, 0.5);
    border-radius: 4px;
    font-size: 12px;
    font-weight: 500;
    color: #6b7280;
    border-bottom: 1px solid #e5e7eb;
  }

  &__waiting-list-row {
    display: grid;
    grid-template-columns: 5fr 2fr 2fr 2fr 1fr;
    gap: 8px;
    padding: 4px 8px;
    background-color: #ffffff;
    border-radius: 4px;
    font-size: 14px;
    align-items: center;
  }

  &__waiting-list-col {
    display: flex;
    justify-content: center;
    align-items: center;
    text-align: center;
    font-size: 12px;
    font-weight: 500;

    &--participants {
      justify-content: flex-start;
      text-align: left;
      flex-direction: column;
      align-items: flex-start;
      gap: 4px;
    }

    &--wish {
      justify-content: center;
    }

    &--wait {
      justify-content: center;
    }
  }

  &__waiting-list-student-name {
    color: #374151;
    font-weight: 400;
  }

  &__waiting-list-student-schedule {
    display: flex;
    gap: 4px;
    font-size: 12px;
    color: #6b7280;
  }

  &__waiting-list-schedule-chip {
    cursor: pointer;
    padding: 2px 4px;
    border-radius: 2px;
    font-weight: 500;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
    }
  }

  &__waiting-list-preferences {
    display: flex;
    gap: 4px;
  }

  &__waiting-list-preference-btn {
    height: 24px;
    padding: 0 8px;
    font-size: 12px;
    background-color: transparent;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    transition: all 0.2s ease;
    cursor: pointer;

    &:hover {
      background-color: #f9fafb;
    }

    &:disabled {
      cursor: default;
    }

    &--current {
      background-color: #dbeafe;
      border-color: #93c5fd;
    }

    &--go-home {
      border-color: #f59e0b;
      background-color: #fef3c7;
    }

    &--locked {
      background-color: #f3f4f6;
      border-color: #d1d5db;
      color: #9ca3af;
      cursor: not-allowed;
    }

    // React-style preference border classes
    &--first-choice {
      border: 2px solid #22c55e;
      background-color: #f0fdf4;
    }

    &--second-choice {
      border: 2px solid #3b82f6;
      background-color: #eff6ff;
    }

    &--third-choice {
      border: 2px solid #f97316;
      background-color: #fff7ed;
    }
  }

  &__waiting-list-wait-btn {
    height: 24px;
    width: 24px;
    padding: 0;
    font-size: 12px;
    background-color: #f3f4f6;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
    }
  }

  &__waiting-list-wait-placeholder {
    height: 24px;
    width: 24px;
  }

  &__waiting-list-empty {
    text-align: center;
    padding: 8px 0;
    font-size: 12px;
    color: #6b7280;
  }

  &__course {
    border-radius: 8px;
    padding: 12px;
    background-color: rgba(243, 244, 246, 0.2);
    margin-bottom: 12px;
    transition: all 0.2s ease;

    &--overfilled {
      background-color: rgba(254, 226, 226, 0.3);
      border: 1px solid #fecaca;
    }

    &--near-full {
      background-color: rgba(254, 243, 199, 0.3);
      border: 1px solid #fde68a;
    }

    &--locked {
      background-color: rgba(249, 250, 251, 0.3);
      border: 1px solid #d1d5db;
    }
  }

  &__course-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 8px;
  }

  &__course-title {
    display: flex;
    align-items: center;
    gap: 8px;
    margin: 0;
    font-size: 14px;
    font-weight: 500;
  }

  &__course-name {
    cursor: pointer;
    background: none;
    border: none;
    padding: 0;
    font-size: 14px;
    font-weight: 500;
    color: #374151;
    transition: color 0.2s ease;

    &:hover {
      color: #3b82f6;
    }
  }

  &__course-capacity {
    font-size: 12px;
    padding: 4px 10px;
    border-radius: 20px;
    font-weight: 600;
    background-color: #6b7280;
    color: #ffffff;
    border: 1px solid #6b7280;

    &.cd-capacity--over {
      background-color: #dc2626;
      border-color: #dc2626;
    }

    &.cd-capacity--near {
      background-color: #ea580c;
      border-color: #ea580c;
    }

    &.cd-capacity--neutral {
      background-color: #6b7280;
      border-color: #6b7280;
    }
  }

  &__course-overfilled-badge {
    font-size: 11px;
    padding: 4px 10px;
    border-radius: 20px;
    font-weight: 600;
    background-color: #dc2626;
    color: #ffffff;
    border: 1px solid #dc2626;
    animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
    white-space: nowrap;
  }

  &__course-search-badge {
    font-size: 12px;
    padding: 4px 10px;
    border-radius: 20px;
    font-weight: 600;
    background-color: transparent;
    color: #6b7280;
    border: 1px solid #d1d5db;
  }

  &__course-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__course-lock-btn {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 4px 8px;
    font-size: 12px;
    background-color: transparent;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s ease;
    color: #6b7280;

    &:hover {
      background-color: #f3f4f6;
      color: #374151;
    }

    &--locked {
      background-color: #dc2626;
      border-color: #dc2626;
      color: #ffffff;

      &:hover {
        background-color: #b91c1c;
        border-color: #b91c1c;
      }
    }
  }

  &__course-lock-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__course-approval-btn {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 4px 8px;
    font-size: 12px;
    background-color: transparent;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s ease;
    color: #6b7280;

    &:hover {
      background-color: #f3f4f6;
      color: #374151;
    }

    &--approved {
      background-color: #dcfce7;
      border-color: #bbf7d0;
      color: #166534;

      &:hover {
        background-color: #bbf7d0;
        border-color: #86efac;
      }
    }
  }

  &__course-approval-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__course-approval-text {
    font-size: 12px;
    font-weight: 400;
  }

  &__course-notes-btn {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
    padding: 0;
    font-size: 12px;
    background-color: transparent;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s ease;
    color: #6b7280;

    &:hover {
      background-color: #f3f4f6;
      color: #374151;
    }
  }

  &__course-notes-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__course-notes-badge {
    position: absolute;
    top: -6px;
    right: -6px;
    display: flex;
    align-items: center;
    justify-content: center;
    min-width: 16px;
    height: 16px;
    padding: 0 4px;
    font-size: 10px;
    font-weight: 500;
    background-color: #dc2626;
    color: #ffffff;
    border-radius: 8px;
    border: 1px solid #ffffff;
  }

  @keyframes pulse {
    0%, 100% {
      opacity: 1;
    }
    50% {
      opacity: 0.5;
    }
  }

  &__course-details {
    margin-bottom: 8px;
  }

  &__course-meta {
    display: flex;
    align-items: center;
    gap: 16px;
    font-size: 12px;
    color: #6b7280;
  }

  &__course-meta-item {
    font-size: 12px;
    color: #6b7280;
  }

  &__course-notes-indicator {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  &__course-notes-badge-small {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 2px 8px;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 400;
    background-color: transparent;
    color: #6b7280;
    border: 1px solid #d1d5db;
  }

  &__course-notes-icon-small {
    width: 8px;
    height: 8px;
    stroke: currentColor;
    fill: none;
  }

  &__course-notes-toggle {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 16px;
    height: 16px;
    padding: 0;
    background-color: transparent;
    border: none;
    cursor: pointer;
    color: #6b7280;
    transition: color 0.2s ease;

    &:hover {
      color: #374151;
    }
  }

  &__course-notes-toggle-icon {
    width: 8px;
    height: 8px;
    stroke: currentColor;
    fill: none;
  }

  &__course-participants {
    display: flex;
    flex-direction: column;
  }

  &__course-participants-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 4px 8px;
    background-color: rgba(243, 244, 246, 0.5);
    border-radius: 4px;
    font-size: 12px;
    font-weight: 500;
    color: #6b7280;
    border-bottom: 1px solid #e5e7eb;
    margin-bottom: 6px;
  }

  &__course-participants-grid {
    display: grid;
    grid-template-columns: repeat(12, 1fr);
    gap: 8px;
    flex: 1;
  }

  &__course-participants-col {
    display: flex;
    justify-content: center;
    align-items: center;
    text-align: center;
    font-size: 12px;
    font-weight: 500;

    &--participants {
      grid-column: span 5;
      justify-content: flex-start;
      text-align: left;
      flex-direction: column;
      align-items: flex-start;
      gap: 4px;
    }

    &--wish {
      grid-column: span 2;
      justify-content: center;
    }

    &--wait {
      grid-column: span 1;
      justify-content: center;
    }
  }

  &__course-participants-toggle-container {
    margin-left: 8px;
  }

  &__course-participants-toggle {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 16px;
    height: 16px;
    padding: 0;
    background-color: transparent;
    border: none;
    cursor: pointer;
    color: #6b7280;
    transition: color 0.2s ease;

    &:hover {
      color: #374151;
    }
  }

  &__course-participants-toggle-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__course-participants-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 4px 8px;
    background-color: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 4px;
    font-size: 14px;
  }

  &__course-participants-student-name {
    color: #374151;
    font-weight: 400;
  }

  &__course-participants-student-schedule {
    display: flex;
    gap: 4px;
    font-size: 12px;
    color: #6b7280;
  }

  &__course-participants-schedule-chip {
    cursor: pointer;
    padding: 2px 4px;
    border-radius: 2px;
    font-weight: 500;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
    }
  }

  &__course-participants-preference-btn {
    height: 24px;
    padding: 0 8px;
    font-size: 12px;
    width: 100%;
    background-color: transparent;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    transition: all 0.2s ease;
    cursor: pointer;

    &:hover {
      background-color: #f9fafb;
    }

    &:disabled {
      cursor: default;
    }

    &--current {
      background-color: #dbeafe;
      border-color: #93c5fd;
    }

    &--go-home {
      border-color: #f59e0b;
      background-color: #fef3c7;
    }

    &--locked {
      background-color: #f3f4f6;
      border-color: #d1d5db;
      color: #9ca3af;
      cursor: not-allowed;
    }

    // React-style preference border classes
    &--first-choice {
      border: 2px solid #22c55e;
      background-color: #f0fdf4;
    }

    &--second-choice {
      border: 2px solid #3b82f6;
      background-color: #eff6ff;
    }

    &--third-choice {
      border: 2px solid #f97316;
      background-color: #fff7ed;
    }
  }

  &__course-participants-wait-btn {
    height: 24px;
    width: 24px;
    padding: 0;
    font-size: 12px;
    background-color: #f3f4f6;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
    }
  }

  &__course-participants-wait-placeholder {
    height: 24px;
    width: 24px;
  }

  &__course-participants-spacer {
    margin-left: 8px;
    width: 16px;
  }

  &__course-participants-empty {
    text-align: center;
    padding: 8px 0;
    font-size: 12px;
    color: #6b7280;
  }

  &__problems {
    margin-bottom: 16px;
    padding: 12px;
    background-color: #fef2f2;
    border-left: 4px solid #f87171;
    border-radius: 8px;
  }

  &__problems-title {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 8px;
    font-size: 14px;
    font-weight: 500;
    color: #991b1b;
  }

  &__problems-icon {
    width: 16px;
    height: 16px;
    stroke: currentColor;
    fill: none;
  }

  &__problems-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__problems-item {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    padding: 8px;
    background-color: #ffffff;
    border: 1px solid #fecaca;
    border-radius: 4px;
  }

  &__problems-content {
    flex: 1;
  }

  &__problems-header {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 4px;
  }

  &__problems-course {
    font-size: 14px;
    font-weight: 500;
    color: #991b1b;
  }

  &__problems-author {
    font-size: 12px;
    color: #dc2626;
  }

  &__problems-date {
    font-size: 12px;
    color: #ef4444;
  }

  &__problems-text {
    font-size: 14px;
    color: #991b1b;
  }

  &__problems-resolve-btn {
    display: flex;
    align-items: center;
    gap: 4px;
    margin-left: 12px;
    height: 24px;
    padding: 0 12px;
    font-size: 12px;
    background-color: #dcfce7;
    color: #166534;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover {
      background-color: #bbf7d0;
    }
  }

  &__problems-resolve-icon {
    width: 12px;
    height: 12px;
    stroke: currentColor;
    fill: none;
  }

  &__title {
    font-size: 20px;
    font-weight: 600;
    margin: 0;
    color: #111827;
  }

  &__subtitle {
    font-size: 14px;
    font-weight: 400;
    margin: 0;
    color: #6b7280;
  }

  &__search-input,
  &__day-select {
    padding: 8px 12px;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 14px;
    background-color: #ffffff;

    &:focus {
      outline: none;
      border-color: #3b82f6;
      box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
  }

  &__loading {
    padding: 32px;
    text-align: center;
    font-size: 14px;
    color: #6b7280;
  }

  &__content {
    display: flex;
    flex-direction: column;
    gap: 24px;
    padding: 24px;
    background-color: #f8fafc;
    min-height: 100vh;
  }

  &__empty {
    padding: 32px;
    text-align: center;
    font-size: 14px;
    color: #6b7280;
    background-color: #ffffff;
    border: 1px dashed #d1d5db;
    border-radius: 12px;
  }
}

/* Modern Card Design */
.cd-card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px 0 rgba(0, 0, 0, 0.06);

  &.course-distribution__section--near-full {
    border-color: #f59e0b;
    box-shadow: 0 1px 3px 0 rgba(245, 158, 11, 0.1), 0 1px 2px 0 rgba(245, 158, 11, 0.06);
  }

  &.course-distribution__section--overfilled {
    border-color: #ef4444;
    box-shadow: 0 1px 3px 0 rgba(239, 68, 68, 0.1), 0 1px 2px 0 rgba(239, 68, 68, 0.06);
  }

  &.course-distribution__section--locked {
    border-color: #9ca3af;
    background-color: #f9fafb;
  }

  &__top {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 20px;
    border-bottom: 1px solid #e5e7eb;
    background-color: #f8fafc;
  }

  &__title {
    display: flex;
    align-items: center;
    gap: 12px;
    margin: 0;
    font-weight: 600;
    color: #111827;
    font-size: 16px;
  }

  &__name {
    background: none;
    border: none;
    color: #111827;
    cursor: pointer;
    padding: 0;
    font-weight: 600;
    font-size: 16px;

    &:hover {
      color: #3b82f6;
    }
  }

  &__underline {
    display: none;
  }
}

/* Meta information */
.cd-meta {
  display: flex;
  gap: 16px;
  color: #6b7280;
  font-size: 12px;
  padding: 0 20px 8px;
  background-color: #f8fafc;
}

/* Modern Capacity Badge */
.cd-capacity {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: 600;
  border: 1px solid transparent;
}

.cd-capacity--neutral {
  background-color: #f1f5f9;
  color: #475569;
  border-color: #e2e8f0;
}

.cd-capacity--normal {
  background-color: #dcfce7;
  color: #166534;
  border-color: #bbf7d0;
}

.cd-capacity--near {
  background-color: #fef3c7;
  color: #92400e;
  border-color: #fde68a;
}

.cd-capacity--over {
  background-color: #fee2e2;
  color: #7f1d1d;
  border-color: #fecaca;
}

.cd-capacity--locked {
  background-color: #f1f5f9;
  color: #64748b;
  border-color: #e2e8f0;
}

/* Status badge */
@keyframes cdPulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.cd-status-badge {
  margin-left: 8px;
  background: #ef4444;
  color: #fff;
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 11px;
  font-weight: 600;
  animation: cdPulse 2s ease-in-out infinite;
  white-space: nowrap;
}

/* Action buttons */
.cd-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.cd-icon-btn {
  height: 32px;
  width: 32px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
  border-radius: 8px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;

  &:hover {
    background-color: #f3f4f6;
    border-color: #d1d5db;
  }
}

.cd-small-btn {
  height: 32px;
  padding: 0 12px;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
  cursor: pointer;
  color: #6b7280;
  transition: all 0.2s ease;

  &:hover {
    background-color: #f3f4f6;
    color: #111827;
    border-color: #d1d5db;
  }
}

.cd-icon {
  width: 16px;
  height: 16px;
  stroke: currentColor;
  fill: none;
  display: block;
  flex-shrink: 0;
}

.cd-icon--blue {
  color: #3b82f6;
}

/* Icon text styling */
.cd-icon-text {
  font-size: 14px;
  line-height: 1;
  display: block;
}

/* Locked button styling */
.cd-icon-btn--locked {
  background-color: #fef3c7 !important;
  border-color: #f59e0b !important;
  color: #92400e !important;

  &:hover {
    background-color: #fde68a !important;
    border-color: #f59e0b !important;
  }
}

/* Locked course styling */
.cd-card.course-distribution__section--locked {
  background-color: #f9fafb;

  .cd-capacity {
    background-color: #f1f5f9;
    color: #64748b;
    border-color: #e2e8f0;
  }
}

.cd-icon-btn--note {
  position: relative;
}

.cd-note-badge {
  position: absolute;
  top: -6px;
  right: -6px;
  background-color: #ef4444;
  color: #ffffff;
  border-radius: 10px;
  width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 600;
}

/* Table-style participants layout */
.cd-participants {
  margin: 0;
  background-color: #ffffff;
}

.cd-participants__bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 20px;
  background-color: #f8fafc;
  border-bottom: 1px solid #e5e7eb;
}

/* Modern grid layout */
.cd-grid {
  display: grid;
  grid-template-columns: 3fr 1.5fr 1.5fr 1.5fr 80px;
  gap: 12px;
  align-items: center;
  padding: 0 20px;
}

.cd-grid--header {
  font-size: 12px;
  font-weight: 600;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.cd-col--person {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.cd-col--center {
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Student rows */
.cd-rows {
  display: flex;
  flex-direction: column;
}

.cd-row {
  border-bottom: 1px solid #f1f5f9;
  padding: 12px 0;
  transition: background-color 0.15s ease;

  &:hover {
    background-color: #f8fafc;
  }

  &:last-child {
    border-bottom: none;
  }
}

/* Student info styling */
.cd-chip {
  background-color: #f1f5f9;
  color: #475569;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
  width: fit-content;
}

.cd-name {
  font-size: 14px;
  color: #111827;
  font-weight: 500;
}

/* Add schedule info styling */
.cd-schedule-info {
  display: flex;
  gap: 6px;
  margin-top: 4px;

  .cd-schedule-chip {
    background-color: #f1f5f9;
    color: #64748b;
    padding: 2px 6px;
    border-radius: 8px;
    font-size: 10px;
    font-weight: 500;
    cursor: pointer;
    transition: background-color 0.15s ease;

    &:hover {
      background-color: #e2e8f0;
    }
  }
}

/* Modern preference buttons */
.cd-pref-btn {
  height: 32px;
  padding: 0 12px;
  font-size: 12px;
  font-weight: 500;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
  color: #374151;
  cursor: pointer;
  width: 100%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  transition: all 0.2s ease;

  &:hover:not(:disabled) {
    background-color: #f3f4f6;
    border-color: #d1d5db;
  }
}

.cd-pref-btn--first {
  background-color: #dcfce7;
  border-color: #bbf7d0;
  color: #166534;

  &:hover:not(:disabled) {
    background-color: #bbf7d0;
  }
}

.cd-pref-btn--second {
  background-color: #dbeafe;
  border-color: #93c5fd;
  color: #1d4ed8;

  &:hover:not(:disabled) {
    background-color: #bfdbfe;
  }
}

.cd-pref-btn--third {
  background-color: #fed7aa;
  border-color: #fdba74;
  color: #ea580c;

  &:hover:not(:disabled) {
    background-color: #fdba74;
  }
}

.cd-pref-btn--current {
  background-color: #f0f9ff;
  border-color: #0ea5e9;
  color: #0c4a6e;
  box-shadow: 0 0 0 3px rgba(14, 165, 233, 0.1);
}

.cd-pref-btn--locked {
  border-color: #d1d5db !important;
  color: #9ca3af !important;
  background-color: #f9fafb !important;
  cursor: not-allowed;
}

.cd-pref-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modern wait button */
.cd-wait-btn {
  height: 32px;
  width: 32px;
  font-size: 12px;
  font-weight: 600;
  background-color: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  cursor: pointer;
  color: #64748b;
  transition: all 0.2s ease;

  &:hover:not(:disabled) {
    background-color: #e2e8f0;
    border-color: #cbd5e1;
    color: #475569;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}

/* Empty state */
.cd-empty-text {
  color: #9ca3af;
  font-size: 13px;
  padding: 20px;
  text-align: center;
  font-style: italic;
}

/* Chevron button for expand/collapse */
.cd-icon-btn--chev {
  border: none;
  background: transparent;

  &:hover {
    background-color: #f3f4f6;
    border: none;
  }
}

/* Tailwind-like utilities for the Problems Section */
.mb-4 { margin-bottom: 1rem; }
.p-3 { padding: 0.75rem; }
.bg-red-50 { background-color: #fef2f2; }
.border-l-4 { border-left-width: 4px; }
.border-red-400 { border-left-color: #f87171; }
.rounded { border-radius: 0.375rem; }
.text-red-800 { color: #991b1b; }
.font-medium { font-weight: 500; }
.flex { display: flex; }
.items-center { align-items: center; }
.gap-2 { gap: 0.5rem; }
.h-4 { height: 1rem; }
.w-4 { width: 1rem; }
.space-y-2 > * + * { margin-top: 0.5rem; }
.items-start { align-items: flex-start; }
.justify-between { justify-content: space-between; }
.bg-white { background-color: #ffffff; }
.p-2 { padding: 0.5rem; }
.border { border-width: 1px; }
.border-red-200 { border-color: #fecaca; }
.flex-1 { flex: 1 1 0%; }
.font-medium { font-weight: 500; }
.text-red-700 { color: #b91c1c; }
.text-xs { font-size: 0.75rem; line-height: 1rem; }
.text-red-600 { color: #dc2626; }
.text-red-500 { color: #ef4444; }
.text-red-800 { color: #991b1b; }
.text-sm { font-size: 0.875rem; line-height: 1.25rem; }
.ml-3 { margin-left: 0.75rem; }
.h-6 { height: 1.5rem; }
.px-3 { padding-left: 0.75rem; padding-right: 0.75rem; }
.py-1 { padding-top: 0.25rem; padding-bottom: 0.25rem; }
.bg-green-100 { background-color: #dcfce7; }
.text-green-700 { color: #15803d; }
.hover\:bg-green-200:hover { background-color: #bbf7d0; }
.border-0 { border-width: 0px; }
.transition-colors { transition-property: color, background-color, border-color, text-decoration-color, fill, stroke; transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1); transition-duration: 150ms; }
</style>
