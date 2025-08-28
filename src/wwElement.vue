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
          @click="autoEnrollStudents"
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
            v-model="selectedSemesterIndex"
            @change="onSemesterChange"
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
            :value="selectedDayIndex"
            @change="onDayChange($event.target.value)"
            class="course-distribution__day-select"
          >
            <option
              v-for="(day, index) in (currentSemesterData?.schedule || [])"
              :key="index"
              :value="index"
            >
              {{ day.day.slice(0, 3) }}
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
          <div v-for="student in waitingListStudents" :key="student.id + '-' + selectedDayIndex" class="course-distribution__waiting-list-row">
            <div class="course-distribution__waiting-list-col course-distribution__waiting-list-col--participants">
              <span class="course-distribution__waiting-list-student-name">{{ student.class }} {{ student.name }}</span>
              <div class="course-distribution__waiting-list-student-schedule">
                                                  <span
                    v-for="(enrollment, dayIndex) in student.enrollments" 
                    :key="dayIndex"
                    class="course-distribution__waiting-list-schedule-chip course-distribution__schedule-chip--clickable"
                    @click="openScheduleModal(student.id, enrollment || 'go-home', dayIndex, getDayName(dayIndex), enrollment || 'go-home', $event)"
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
                     :title="isLockedTarget(student.first_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.first_choice)"
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
                     :title="isLockedTarget(student.second_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.second_choice)"
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
                     :title="isLockedTarget(student.third_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.third_choice)"
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
                        class="course-distribution__going-home-schedule-chip course-distribution__schedule-chip--clickable"
                        @click="openScheduleModal(student.id, enrollment || 'go-home', dayIndex, getDayName(dayIndex), enrollment || 'go-home', $event)"
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
                          class="course-distribution__going-home-schedule-chip course-distribution__schedule-chip--clickable"
                          @click="openScheduleModal(student.id, enrollment || 'go-home', dayIndex, getDayName(dayIndex), enrollment || 'go-home')"
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
                             student.first_choice === 'go-home' ? 'course-distribution__going-home-preference-btn--go-home' : '',
                             isLockedTarget(student.first_choice) ? 'course-distribution__going-home-preference-btn--locked' : ''
                           ]"
                                                      :disabled="isCurrentChoice(student, 1) || isLockedTarget(student.first_choice)"
                            @click="!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice) && onStudentMove(student.id, student.first_choice)"
                                                      :title="isLockedTarget(student.first_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.first_choice)"
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
                             student.second_choice === 'go-home' ? 'course-distribution__going-home-preference-btn--go-home' : '',
                             isLockedTarget(student.second_choice) ? 'course-distribution__going-home-preference-btn--locked' : ''
                           ]"
                                                      :disabled="isCurrentChoice(student, 2) || isLockedTarget(student.second_choice)"
                            @click="!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice) && onStudentMove(student.id, student.second_choice)"
                                                      :title="isLockedTarget(student.second_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.second_choice)"
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
                             student.third_choice === 'go-home' ? 'course-distribution__going-home-preference-btn--go-home' : '',
                             isLockedTarget(student.third_choice) ? 'course-distribution__going-home-preference-btn--locked' : ''
                           ]"
                                                      :disabled="isCurrentChoice(student, 3) || isLockedTarget(student.third_choice)"
                            @click="!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice) && onStudentMove(student.id, student.third_choice)"
                                                      :title="isLockedTarget(student.third_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.third_choice)"
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
         course.forcedFull ? 'course-distribution__course--locked' : ''
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
               v-if="course.forcedFull"
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
              <span v-if="course.notes && course.notes.length > 0" class="course-distribution__course-notes-badge">{{ course.notes.length }}</span>
            </button>
          </div>
        </div>

        <!-- Course Details Section - Only show when expanded -->
        <div v-if="isCourseDetailsVisible(course.id)" class="course-distribution__course-details">
          <div class="course-distribution__course-meta">
            <span v-if="course.teacher" class="course-distribution__course-meta-item">Teacher: {{ course.teacher }}</span>
            <span v-if="course.room" class="course-distribution__course-meta-item">Room: {{ course.room }}</span>
            <span v-if="course.availableGrades" class="course-distribution__course-meta-item">Grades: {{ course.availableGrades.join(', ') }}</span>
            <div v-if="course.notes && course.notes.length > 0" class="course-distribution__course-notes-indicator">
              <span class="course-distribution__course-notes-badge-small">
                <svg class="course-distribution__course-notes-icon-small" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z" />
                </svg>
                {{ course.notes.length }} note{{ course.notes.length > 1 ? 's' : '' }}
                <svg v-if="course.notes.some(note => note.isProblem && !note.isResolved)" class="course-distribution__course-notes-problem-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.732-.833-2.464 0L4.35 16.5c-.77.833.192 2.5 1.732 2.5z" />
                </svg>
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

        <!-- Notes Section - Only show when course details are visible and notes are toggled on -->
        <div v-if="isCourseDetailsVisible(course.id) && isNotesVisible(course.id)" class="course-distribution__course-notes">
          <div v-if="course.notes && course.notes.length > 0" class="course-distribution__course-notes-list">
            <div v-for="note in course.notes" :key="note.id" class="course-distribution__course-note" :class="{
              'course-distribution__course-note--problem': note.isProblem && !note.isResolved,
              'course-distribution__course-note--resolved': note.isResolved,
              'course-distribution__course-note--normal': !note.isProblem && !note.isResolved
            }">
              <div class="course-distribution__course-note-header">
                <div class="course-distribution__course-note-meta">
                  <span class="course-distribution__course-note-author">{{ note.author }}</span>
                  <span class="course-distribution__course-note-timestamp">
                    <svg class="course-distribution__course-note-clock-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
                    </svg>
                    {{ formatTimestamp(note.timestamp) }}
                  </span>
                  <span v-if="note.isProblem && !note.isResolved" class="course-distribution__course-note-problem-badge">Problem</span>
                  <span v-if="note.isResolved" class="course-distribution__course-note-resolved-badge">Resolved</span>
                </div>
                <div v-if="!(editingNote?.courseId === course.id && editingNote?.noteId === note.id)" class="course-distribution__course-note-actions">
                  <button
                    class="course-distribution__course-note-action-btn"
                    :class="{ 'course-distribution__course-note-action-btn--problem': note.isProblem }"
                    @click="toggleNoteProblem(course.id, note.id)"
                    :title="note.isProblem ? 'Remove from problems' : 'Flag as problem'"
                  >
                    <svg class="course-distribution__course-note-action-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.732-.833-2.464 0L4.35 16.5c-.77.833.192 2.5 1.732 2.5z" />
                    </svg>
                  </button>
                  <button
                    v-if="note.isProblem && !note.isResolved"
                    class="course-distribution__course-note-action-btn course-distribution__course-note-action-btn--resolve"
                    @click="toggleProblemResolved(course.id, note.id)"
                    title="Mark as resolved"
                  >
                    <svg class="course-distribution__course-note-action-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                    </svg>
                  </button>
                  <button
                    class="course-distribution__course-note-action-btn course-distribution__course-note-action-btn--edit"
                    @click="startEditNote(course.id, note.id, note.text)"
                    title="Edit note"
                  >
                    <svg class="course-distribution__course-note-action-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
                    </svg>
                  </button>
                  <button
                    class="course-distribution__course-note-action-btn course-distribution__course-note-action-btn--delete"
                    @click="confirmDelete(course.id, note.id)"
                    title="Delete note"
                  >
                    <svg class="course-distribution__course-note-action-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                    </svg>
                  </button>
                </div>
              </div>
              <div v-if="editingNote?.courseId === course.id && editingNote?.noteId === note.id" class="course-distribution__course-note-edit">
                <div class="course-distribution__course-note-edit-form">
                  <input
                    v-model="editNoteText"
                    class="course-distribution__course-note-edit-input"
                    @keydown="(e) => {
                      if (e.key === 'Enter') saveEditNote(course.id, note.id);
                      if (e.key === 'Escape') cancelEditNote();
                    }"
                    autofocus
                  />
                  <button
                    class="course-distribution__course-note-action-btn course-distribution__course-note-action-btn--save"
                    @click="saveEditNote(course.id, note.id)"
                    title="Save changes"
                  >
                    <svg class="course-distribution__course-note-action-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                    </svg>
                  </button>
                  <button
                    class="course-distribution__course-note-action-btn course-distribution__course-note-action-btn--cancel"
                    @click="cancelEditNote"
                    title="Cancel editing"
                  >
                    <svg class="course-distribution__course-note-action-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                  </button>
                </div>
              </div>
              <div v-else class="course-distribution__course-note-text">
                {{ note.text }}
              </div>
            </div>
          </div>

          <!-- Add Note Form -->
          <div v-if="showAddNote[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`]" class="course-distribution__course-note-add">
            <div class="course-distribution__course-note-add-form">
              <input
                v-model="newNoteText[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`]"
                placeholder="Add a note..."
                class="course-distribution__course-note-add-input"
                @keydown="(e) => {
                  if (e.key === 'Enter') addNote(course.id, newNoteText[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`]);
                  if (e.key === 'Escape') {
                    showAddNote[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`] = false;
                    newNoteText[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`] = '';
                  }
                }"
                autofocus
              />
              <button
                class="course-distribution__course-note-add-btn"
                @click="addNote(course.id, newNoteText[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`])"
              >
                Add
              </button>
              <button
                class="course-distribution__course-note-cancel-btn"
                @click="() => {
                  showAddNote[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`] = false;
                  newNoteText[`${selectedSemesterIndex}-${selectedDayIndex}-${course.id}`] = '';
                }"
              >
                Cancel
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
              <div v-for="student in (enrolledByCourseId[course.id] || [])" :key="student.id + '-' + selectedDayIndex"
               class="course-distribution__course-participants-row">
               <div class="course-distribution__course-participants-grid">
                 <div class="course-distribution__course-participants-col course-distribution__course-participants-col--participants">
                   <span class="course-distribution__course-participants-student-name">{{ student.class }} {{ student.name }}</span>
                   <div class="course-distribution__course-participants-student-schedule">
                                           <span
                        v-for="(enrollment, dayIndex) in student.enrollments" 
                        :key="dayIndex"
                        class="course-distribution__course-participants-schedule-chip course-distribution__schedule-chip--clickable"
                        @click="openScheduleModal(student.id, enrollment || 'go-home', dayIndex, getDayName(dayIndex), enrollment || 'go-home', $event)"
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
                     @click="handleFirstChoiceClick(student)"
                      :title="isLockedTarget(student.first_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.first_choice)"
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
                     @click="handleSecondChoiceClick(student)"
                      :title="isLockedTarget(student.second_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.second_choice)"
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
                     @click="handleThirdChoiceClick(student)"
                      :title="isLockedTarget(student.third_choice) ? 'Course is marked as closed' : formatPreferenceDisplay(student.third_choice)"
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

  <!-- Schedule Popover -->
  <div v-if="activeScheduleModal" class="course-distribution__popover-overlay" @click="closeScheduleModal">
    <div 
      class="course-distribution__popover" 
      @click.stop
      :style="{
        left: popoverPosition.x + 'px',
        top: popoverPosition.y + 'px',
        transform: 'translateX(-50%)'
      }"
    >
      <div class="course-distribution__modal-content">
                  <p class="course-distribution__modal-day-title">
            {{ activeScheduleModal.day }} - {{ activeScheduleModal.activity === 'go-home' ? 'Going Home' : getCourseName(activeScheduleModal.activity, activeScheduleModal.dayIndex) }}
          </p>
        
        <!-- Current Choice Info -->
        <div v-if="getCurrentChoice(activeScheduleModal.childId, activeScheduleModal.activity)" class="course-distribution__modal-current-choice">
          <p class="course-distribution__modal-current-text">
            Current: {{ getCurrentChoice(activeScheduleModal.childId, activeScheduleModal.activity).choice }} choice
          </p>
        </div>
        
        <!-- Student's Preferences -->
        <div class="course-distribution__modal-preferences">
          <p class="course-distribution__modal-preferences-title">Student's preferences:</p>
          <div class="course-distribution__modal-preferences-list">
            <div 
              v-for="pref in getChildPreferences(activeScheduleModal.childId, activeScheduleModal.dayIndex)" 
              :key="pref.choice"
              class="course-distribution__modal-preference-item"
              :class="{ 'course-distribution__modal-preference-item--active': pref.courseId === activeScheduleModal.activity }"
            >
              <span class="course-distribution__modal-preference-label">{{ pref.choice }}:</span>
              <button 
                class="course-distribution__modal-preference-badge"
                :class="{ 
                  'course-distribution__modal-preference-badge--active': pref.courseId === activeScheduleModal.activity,
                  'course-distribution__modal-preference-badge--clickable': pref.courseId !== 'go-home'
                }"
                @click="pref.courseId !== 'go-home' && openPopoverForm(activeScheduleModal.childId, pref.courseId, activeScheduleModal.dayIndex)"
              >
                {{ pref.name }}
                <span v-if="pref.courseId !== 'go-home'" class="course-distribution__modal-preference-capacity">
                  {{ (() => {
                    const day = currentSemesterData?.schedule[activeScheduleModal.dayIndex];
                    const course = day?.courses.find(c => c.id === pref.courseId);
                    return course ? `${course.enrolledChildren.length}/${course.maxCapacity}` : '';
                  })() }}
                </span>
              </button>
            </div>
          </div>
        </div>

        <!-- Nested Inline Form (like React's renderPopoverInlineForm) -->
        <div v-if="activePopoverForm && activePopoverForm.childId === activeScheduleModal.childId && activePopoverForm.dayIndex === activeScheduleModal.dayIndex" class="course-distribution__modal-inline-form">
          <div class="course-distribution__modal-inline-form-header">
            <div class="course-distribution__modal-inline-form-title">
              <span class="course-distribution__modal-inline-form-course-name">{{ getCourseName(activePopoverForm.courseId, activePopoverForm.dayIndex) }}</span>
              <span v-if="activePopoverForm.courseId !== 'go-home'" class="course-distribution__modal-inline-form-badge">
                {{ (() => {
                  const day = currentSemesterData?.schedule[activePopoverForm.dayIndex];
                  const course = day?.courses.find(c => c.id === activePopoverForm.courseId);
                  const enrolledCount = course?.enrolledChildren.length || 0;
                  return `${enrolledCount}/${course?.maxCapacity || 0}`;
                })() }}
              </span>
            </div>
            <button class="course-distribution__modal-inline-form-close" @click="closePopoverForm">
              <svg class="course-distribution__modal-inline-form-close-icon" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>

          <!-- Currently enrolled students in the target course -->
          <div v-if="activePopoverForm.courseId !== 'go-home' && enrolledByCourseId[activePopoverForm.courseId]?.length > 0" class="course-distribution__modal-inline-form-enrolled">
            <p class="course-distribution__modal-inline-form-enrolled-title">Currently enrolled:</p>
            <div class="course-distribution__modal-inline-form-enrolled-list">
              <div 
                v-for="enrolledStudent in enrolledByCourseId[activePopoverForm.courseId]" 
                :key="enrolledStudent.id"
                class="course-distribution__modal-inline-form-enrolled-item"
              >
                <span class="course-distribution__modal-inline-form-enrolled-name">{{ enrolledStudent.class }} {{ enrolledStudent.name }}</span>
                
                <!-- Move dropdown for enrolled students -->
                <select 
                  v-if="getAvailableCoursesForMove(activePopoverForm.dayIndex, activePopoverForm.courseId).length > 0"
                  class="course-distribution__modal-inline-form-move-select"
                  @change="onStudentMove(enrolledStudent.id, $event.target.value, activePopoverForm.dayIndex)"
                >
                  <option value="">Move</option>
                  <option 
                    v-for="course in getAvailableCoursesForMove(activePopoverForm.dayIndex, activePopoverForm.courseId)"
                    :key="course.id"
                    :value="course.id"
                  >
                    {{ course.name }}
                    <span v-if="course.spotsAvailable !== null">({{ course.spotsAvailable }})</span>
                  </option>
                </select>
              </div>
            </div>
          </div>

          <div v-else-if="activePopoverForm.courseId !== 'go-home'" class="course-distribution__modal-inline-form-no-students">
            <p class="course-distribution__modal-inline-form-no-students-text">No children enrolled</p>
          </div>

          <!-- Action Button -->
          <button 
            class="course-distribution__modal-inline-form-action-btn"
            :class="{ 
              'course-distribution__modal-inline-form-action-btn--destructive': getMovementBlockingReason(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex),
              'course-distribution__modal-inline-form-action-btn--default': !getMovementBlockingReason(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex)
            }"
            @click="(() => {
              const blockingReason = getMovementBlockingReason(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex);
              if (blockingReason) {
                alert(`Cannot move child: ${blockingReason}`);
              } else {
                onStudentMove(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex);
                closePopoverForm();
              }
            })()"
            :title="getMovementBlockingReason(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex) || ''"
          >
            <span v-if="getMovementBlockingReason(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex)">
              ⚠️ {{ getMovementBlockingReason(activeScheduleModal.childId, activePopoverForm.courseId, activePopoverForm.dayIndex) }}
            </span>
            <span v-else>
              {{ activePopoverForm.courseId === 'go-home' ? 'Send Home' : `Move Here (${getCourseAvailableSpotsForDay(activePopoverForm.dayIndex, activePopoverForm.courseId)} free)` }}
            </span>
          </button>
        </div>

        <!-- Course Details (if not go-home) -->
        <div v-if="activeScheduleModal.activity !== 'go-home'" class="course-distribution__modal-course-details">
          <p class="course-distribution__modal-course-title">
            Other students enrolled: {{ getCourseName(activeScheduleModal.activity, activeScheduleModal.dayIndex) }}
            <span v-if="getCurrentChoice(activeScheduleModal.childId, activeScheduleModal.activity)">
              ({{ getCurrentChoice(activeScheduleModal.childId, activeScheduleModal.activity).choice }} choice)
            </span>
          </p>
          
          <div v-if="enrolledByCourseId[activeScheduleModal.activity]?.length > 0" class="course-distribution__modal-enrolled-list">
            <div 
              v-for="enrolledStudent in enrolledByCourseId[activeScheduleModal.activity]" 
              :key="enrolledStudent.id"
              class="course-distribution__modal-enrolled-item"
            >
              <span class="course-distribution__modal-enrolled-name">{{ enrolledStudent.class }} {{ enrolledStudent.name }}</span>
            </div>
          </div>
          <p v-else class="course-distribution__modal-no-students">No other students</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { computed, ref, onMounted, nextTick, triggerRef, reactive, toRefs } from 'vue';

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
      /* wwEditor:start */ 
      const editing = props.wwEditorState?.isEditing || false;
      return editing; 
      /* wwEditor:end */
      return false;
    });

    // State
    const loading = ref(true);
    const searchQuery = ref('');
    const selectedDayIndex = ref(0);
    const selectedSemesterIndex = ref(0);
    const daysOfWeek = ref([]);
    const semesters = ref([]);
    const currentSemester = ref('');
    
    // Test counter for debugging

          const visibleParticipants = ref({});
      const collapsedCourseDetails = ref({});
      const isGoingHomeExpanded = ref(true);
      const isGoingHomeChildrenVisible = ref(true);
            const notesVisible = ref({});
      const approvedCourses = ref({});
      
      // Notes functionality state
      const newNoteText = ref({});
      const showAddNote = ref({});
      const editingNote = ref(null); // { courseId, noteId }
      const editNoteText = ref('');
      
      // Popover state for schedule popover
      const activeScheduleModal = ref(null); // { childId, courseId, dayIndex, day, activity }
      const activePopoverForm = ref(null); // { childId, courseId, dayIndex } - for nested form
      const popoverPosition = ref({ x: 0, y: 0 }); // Position for the popover


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
            { id: 'student-1', name: 'Anna Meier', class: '3A', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'course-3', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-2', name: 'Max Schmidt', class: '4B', first_choice: 'course-2', second_choice: 'course-1', third_choice: 'course-3', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-3', name: 'Lea Müller', class: '5A', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'course-2', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-4', name: 'Tom Weber', class: '3B', first_choice: 'course-1', second_choice: 'course-3', third_choice: 'course-2', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-5', name: 'Lisa Fischer', class: '4A', first_choice: 'course-2', second_choice: 'course-1', third_choice: 'course-3', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-6', name: 'Paul Wagner', class: '5B', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'course-3', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-7', name: 'Emma Schulz', class: '3A', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'go-home', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-8', name: 'Felix Klein', class: '4B', first_choice: 'course-1', second_choice: 'course-2', third_choice: 'go-home', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-9', name: 'Nina Hoffmann', class: '5A', first_choice: 'course-2', second_choice: 'course-3', third_choice: 'course-1', enrollments: { 0: undefined, 1: undefined, 2: undefined } },
            { id: 'student-10', name: 'Lukas Meyer', class: '3B', first_choice: 'course-3', second_choice: 'course-1', third_choice: 'course-2', enrollments: { 0: undefined, 1: undefined, 2: undefined } }
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





      currentSemester.value = semesters.value[0].name;
      loading.value = false;
    };

    // Computed properties
    const currentSemesterData = computed(() => {
      // Force reactivity by explicitly accessing the index
      const index = selectedSemesterIndex.value;
      const result = semesters.value[index] || semesters.value[0];
      console.log('currentSemesterData computed:', result?.name, 'index:', index, 'total semesters:', semesters.value.length);
      return result;
    });

    const currentDayData = computed(() => {
      const semester = currentSemesterData.value;
      return semester?.schedule[selectedDayIndex.value] || semester?.schedule[0];
    });

    const selectedDayName = computed(() => {
      const day = currentDayData.value;
      return day?.day || 'Monday';
    });

    const filteredStudents = computed(() => {
      const students = currentSemesterData.value?.children || [];
      console.log('filteredStudents computed:', students.length, 'students for semester:', currentSemesterData.value?.name);
      if (!searchQuery.value.trim()) return students;
      const query = searchQuery.value.toLowerCase();
      return students.filter(student => 
        student.name.toLowerCase().includes(query)
      );
    });

    const filteredCourses = computed(() => {
      const result = currentDayData.value?.courses || [];
      console.log('filteredCourses computed:', result.length, 'courses');
      return result;
    });

    const waitingListStudents = computed(() => {
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      const enrolledIds = currentDayData.value?.courses.flatMap(course => course.enrolledChildren) || [];
      const result = filteredStudents.value.filter(student => 
        !enrolledIds.includes(student.id) && 
        student.name.toLowerCase().includes(searchQuery.value.toLowerCase()) &&
        // Exclude children who are intentionally going home (have go-home preferences or explicitly set to go home)
        !(student.enrollments[dayIndex] === undefined && hasGoHomePreference(student))
      );
      console.log('waitingListStudents computed:', result.length, 'students for semester:', semesterIndex, 'day:', dayIndex);
      return result;
    });

    const goingHomeStudents = computed(() => {
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      const enrolledIds = currentDayData.value?.courses.flatMap(course => course.enrolledChildren) || [];
      return filteredStudents.value.filter(student => 
        !enrolledIds.includes(student.id) && 
        student.name.toLowerCase().includes(searchQuery.value.toLowerCase()) &&
        // Include children who are intentionally going home (have go-home preferences or explicitly set to go home)
        (student.enrollments[dayIndex] === undefined && hasGoHomePreference(student))
      );
    });

    const firstChoiceGoHomeStudents = computed(() => {
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      return filteredStudents.value.filter(student => 
        student.enrollments && student.enrollments[dayIndex] === undefined &&
        student.first_choice === 'go-home'
      );
    });

    const secondOrThirdChoiceGoHomeStudents = computed(() => {
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      return filteredStudents.value.filter(student => 
        student.enrollments && student.enrollments[dayIndex] === undefined &&
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
      
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      courses.forEach(course => {
        enrolled[course.id] = course.enrolledChildren.map(childId => 
          students.find(student => student.id === childId)
        ).filter(Boolean);
      });
      return enrolled;
    });

    const totalEnrolledForDay = computed(() => {
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      const enrolledIds = currentDayData.value?.courses.flatMap(course => course.enrolledChildren) || [];
      const result = enrolledIds.length;
      console.log('totalEnrolledForDay computed:', result, 'for semester:', semesterIndex, 'day:', dayIndex);
      return result;
    });

    const unresolvedProblems = computed(() => {
      const allNotes = [];
      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          day.courses.forEach(course => {
            if (course.notes) {
              course.notes.forEach(note => {
                if (note.isProblem && !note.isResolved) {
                  allNotes.push({
                    ...note,
                    course_id: course.id,
                    created_at: note.timestamp
                  });
                }
              });
            }
          });
        });
      }
      return allNotes;
    });

    // Helper functions
    const getCourseName = (courseId, dayIndex = null) => {
      if (courseId === 'go-home') return 'Go Home';
      
      // If dayIndex is provided, get course from that specific day
      if (dayIndex !== null) {
        const day = currentSemesterData.value?.schedule[dayIndex];
        const course = day?.courses.find(c => c.id === courseId);
        return course?.name || courseId;
      }
      
      // Otherwise, get from current day
      const course = currentDayData.value?.courses.find(c => c.id === courseId);
      return course?.name || courseId;
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

    const formatTimestamp = (timestamp) => {
      if (!timestamp) return '';
      const date = new Date(timestamp);
      return date.toLocaleDateString() + ' ' + date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    };





    const formatPreferenceDisplay = (courseId, dayIndex = null) => {
      if (courseId === 'go-home') return 'Home';
      
      // If dayIndex is provided, get course from that specific day
      if (dayIndex !== null) {
        const day = currentSemesterData.value?.schedule[dayIndex];
        const course = day?.courses.find(c => c.id === courseId);
        return course?.name || courseId;
      }
      
      // Otherwise, get from current day
      const course = filteredCourses.value.find(c => c.id === courseId);
      return course?.name || courseId;
    };

    const isCurrentChoice = (student, choiceNumber) => {
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      const choice = choiceNumber === 1 ? student.first_choice :
                   choiceNumber === 2 ? student.second_choice : 
                   student.third_choice;
      return student.enrollments && student.enrollments[dayIndex] === choice;
    };

    const isLockedTarget = (courseId) => {
      if (!courseId || courseId === 'go-home') return false;
      const course = currentDayData.value?.courses.find(c => c.id === courseId);
      return !!course?.forcedFull;
    };

    const displayRosterCount = (course) => {
      return enrolledByCourseId.value[course.id]?.length || 0;
    };

    const getCourseStatus = (course) => {
      const enrolled = displayRosterCount(course);
      const capacity = course.maxCapacity || course.max_capacity;
      
      if (course.forcedFull) return 'locked';
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
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      return visibleParticipants.value[courseKey] !== false; // Default to true if not set
    };

    const toggleParticipants = (courseId) => {
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      if (visibleParticipants.value[courseKey] === false) {
        visibleParticipants.value[courseKey] = true;
      } else {
        visibleParticipants.value[courseKey] = false;
      }
      // Force reactivity by reassigning the entire object
      visibleParticipants.value = { ...visibleParticipants.value };
    };

    const toggleCourseDetails = (courseId) => {
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      if (collapsedCourseDetails.value[courseKey] === false) {
        collapsedCourseDetails.value[courseKey] = true;
      } else {
        collapsedCourseDetails.value[courseKey] = false;
      }
      // Force reactivity by reassigning the entire object
      collapsedCourseDetails.value = { ...collapsedCourseDetails.value };
    };

    const isCourseDetailsVisible = (courseId) => {
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      return collapsedCourseDetails.value[courseKey] !== false; // Default to true (visible)
    };

    const toggleCourseDetailsVisibility = (courseId) => {
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      if (collapsedCourseDetails.value[courseKey] === false) {
        collapsedCourseDetails.value[courseKey] = true;
      } else {
        collapsedCourseDetails.value[courseKey] = false;
      }
      // Force reactivity by reassigning the entire object
      collapsedCourseDetails.value = { ...collapsedCourseDetails.value };
    };

    const isNotesVisible = (courseId) => {
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      return notesVisible.value[courseKey] !== false; // Default to true if not set
    };

    const toggleNotesVisibility = (courseId) => {
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      if (notesVisible.value[courseKey] === false) {
        notesVisible.value[courseKey] = true;
      } else {
        notesVisible.value[courseKey] = false;
      }
      // Force reactivity by reassigning the entire object
      notesVisible.value = { ...notesVisible.value };
    };

    const isCourseApproved = (courseId) => {
      const approvalKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      return approvedCourses.value[approvalKey] || false;
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
      selectedDayIndex.value = parseInt(day, 10);
      emit('trigger-event', { name: 'dayChange', event: { value: selectedDayIndex.value } });
    };

    const onSearchChange = (query) => {
      if (isEditing.value) return;
      searchQuery.value = query;
      emit('trigger-event', { name: 'searchChange', event: { value: query } });
    };

    const onStudentMove = async (studentId, targetId, targetDayIndex = null) => {
      if (isEditing.value) return;
      console.log('Moving student', studentId, 'to', targetId, 'on day', targetDayIndex);
      
      // Use targetDayIndex if provided, otherwise use current day
      const dayIndex = targetDayIndex !== null ? targetDayIndex : selectedDayIndex.value;
      
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      
      const semester = currentSemesterData.value;
      const day = currentDayData.value;
      const student = semester.children.find(s => s.id === studentId);
      
      if (student) {
        // Remove student from current course
        day.courses.forEach(course => {
          course.enrolledChildren = course.enrolledChildren.filter(id => id !== studentId);
        });
        
        // Add student to new course or set to undefined
        if (targetId === 'waiting') {
          student.enrollments[dayIndex] = undefined;
        } else if (targetId === 'go-home') {
          student.enrollments[dayIndex] = undefined;
        } else {
          const targetCourse = day.courses.find(c => c.id === targetId);
          if (targetCourse) {
            targetCourse.enrolledChildren.push(studentId);
            student.enrollments[dayIndex] = targetId;
          }
        }
      }
      
      emit('trigger-event', { name: 'studentMove', event: { studentId, courseId: targetId, dayIndex } });
    };

    const onLockToggle = (courseId, lockState) => {
      if (isEditing.value) return;
      console.log('Toggling forcedFull for course', courseId, 'to', lockState);
      
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      // Update course forcedFull state in current day data
      const day = currentDayData.value;
      const course = day?.courses.find(c => c.id === courseId);
      if (course) {
        course.forcedFull = lockState;
      }
      
      emit('trigger-event', { name: 'courseLockToggle', event: { courseId, lockState } });
    };

    const onApproveClick = (courseId) => {
      if (isEditing.value) return;
      console.log('Approving course', courseId);
      const approvalKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      if (approvedCourses.value[approvalKey]) {
        delete approvedCourses.value[approvalKey];
      } else {
        approvedCourses.value[approvalKey] = true;
      }
      // Force reactivity by reassigning the entire object
      approvedCourses.value = { ...approvedCourses.value };
      
      emit('trigger-event', { name: 'courseApprove', event: { courseId } });
    };

    // Notes functionality
    const addNote = (courseId, text) => {
      if (!text.trim()) return;
      
      const newNote = {
        id: `note-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
        text: text.trim(),
        author: 'Admin', // In a real app, this would come from user authentication
        timestamp: new Date(),
        isProblem: false,
        isResolved: false
      };

      // Update course notes in current semester data
      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          const course = day.courses.find(c => c.id === courseId);
          if (course) {
            if (!course.notes) course.notes = [];
            course.notes.push(newNote);
          }
        });
      }

      // Clear the input
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      newNoteText.value[courseKey] = '';
      showAddNote.value[courseKey] = false;
      // Force reactivity by reassigning the entire objects
      newNoteText.value = { ...newNoteText.value };
      showAddNote.value = { ...showAddNote.value };
      
      // Force reactivity by reassigning the entire semesters object
      semesters.value = [...semesters.value];
      
      emit('trigger-event', { name: 'noteAdded', event: { courseId, note: newNote } });
    };

    const toggleNoteProblem = (courseId, noteId) => {
      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          const course = day.courses.find(c => c.id === courseId);
          if (course && course.notes) {
            const note = course.notes.find(n => n.id === noteId);
            if (note) {
              note.isProblem = !note.isProblem;
              note.isResolved = note.isProblem ? false : note.isResolved;
            }
          }
        });
      }
      
      // Force reactivity by reassigning the entire semesters object
      semesters.value = [...semesters.value];
      
      emit('trigger-event', { name: 'noteProblemToggled', event: { courseId, noteId } });
    };

    const toggleProblemResolved = (courseId, noteId) => {
      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          const course = day.courses.find(c => c.id === courseId);
          if (course && course.notes) {
            const note = course.notes.find(n => n.id === noteId);
            if (note) {
              note.isResolved = !note.isResolved;
            }
          }
        });
      }
      
      // Force reactivity by reassigning the entire semesters object
      semesters.value = [...semesters.value];
      
      emit('trigger-event', { name: 'problemResolved', event: { courseId, noteId } });
    };

    const confirmDelete = (courseId, noteId) => {
      // Safe confirmation that works in all environments
      try {
        if (typeof window !== 'undefined' && window.confirm) {
          if (window.confirm('Are you sure you want to delete this note?')) {
            deleteNote(courseId, noteId);
          }
        } else {
          // Fallback: just delete without confirmation
          deleteNote(courseId, noteId);
        }
      } catch (error) {
        // If confirmation fails, just delete
        deleteNote(courseId, noteId);
      }
    };

    const deleteNote = (courseId, noteId) => {
      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          const course = day.courses.find(c => c.id === courseId);
          if (course && course.notes) {
            course.notes = course.notes.filter(note => note.id !== noteId);
          }
        });
      }
      
      // Force reactivity by reassigning the entire semesters object
      semesters.value = [...semesters.value];
      
      emit('trigger-event', { name: 'noteDeleted', event: { courseId, noteId } });
    };

    const startEditNote = (courseId, noteId, currentText) => {
      editingNote.value = { courseId, noteId };
      editNoteText.value = currentText;
    };

    const cancelEditNote = () => {
      editingNote.value = null;
      editNoteText.value = '';
    };

    const saveEditNote = (courseId, noteId) => {
      if (!editNoteText.value.trim()) return;

      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          const course = day.courses.find(c => c.id === courseId);
          if (course && course.notes) {
            const note = course.notes.find(n => n.id === noteId);
            if (note) {
              note.text = editNoteText.value.trim();
            }
          }
        });
      }

      editingNote.value = null;
      editNoteText.value = '';
      
      // Force reactivity by reassigning the entire semesters object
      semesters.value = [...semesters.value];
      
      emit('trigger-event', { name: 'noteEdited', event: { courseId, noteId } });
    };

    const onNotesClick = (courseId) => {
      if (isEditing.value) return;
      
      // First, ensure course details are visible
      const courseKey = `${selectedSemesterIndex.value}-${selectedDayIndex.value}-${courseId}`;
      
      if (!isCourseDetailsVisible(courseId)) {
        toggleCourseDetailsVisibility(courseId);
      }
      
      // Then ensure notes are visible
      if (!isNotesVisible(courseId)) {
        toggleNotesVisibility(courseId);
      }
      
      // Show add note form
      showAddNote.value[courseKey] = true;
      
      // Force reactivity by reassigning the entire object
      showAddNote.value = { ...showAddNote.value };
      
      emit('trigger-event', { name: 'courseNotesOpen', event: { courseId } });
    };

    const onResolveNote = async (noteId) => {
      if (isEditing.value) return;
      console.log('Resolving note', noteId);
      
      // Update note in current semester data
      const semester = currentSemesterData.value;
      if (semester?.schedule) {
        semester.schedule.forEach(day => {
          day.courses.forEach(course => {
            if (course.notes) {
              const note = course.notes.find(n => n.id === noteId);
      if (note) {
                note.isResolved = true;
                note.resolvedAt = new Date().toISOString();
              }
            }
          });
        });
      }
      
      emit('trigger-event', { name: 'problemResolved', event: { noteId } });
    };

    const autoEnrollStudents = () => {
      console.log('Auto-enrolling students to FIRST CHOICES only');
      
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      const semester = currentSemesterData.value;
      const day = currentDayData.value;
      
      // Clear all enrollments for current day
      semester.children.forEach(student => {
        if (!student.enrollments) student.enrollments = {};
        student.enrollments[dayIndex] = undefined;
      });
      
      // Clear all course enrollments for current day
      day.courses.forEach(course => {
        course.enrolledChildren = [];
      });
      
      // Auto-enroll students based on their FIRST CHOICE only (like React)
      semester.children.forEach(student => {
        const firstChoice = student.first_choice;
        
        if (firstChoice === 'go-home') {
          student.enrollments[dayIndex] = undefined;
          return;
        }
        
        // Check if child is eligible for their first choice
        const course = day.courses.find(c => c.id === firstChoice);
        if (course && !course.forcedFull) {
          // Always enroll in first choice, even if it goes over capacity (like React)
          course.enrolledChildren.push(student.id);
          student.enrollments[dayIndex] = firstChoice;
        } else {
          // Course is forced full or not found, send home
          student.enrollments[dayIndex] = undefined;
        }
      });
      
      emit('trigger-event', { name: 'autoEnrollStudents' });
    };

    // Simplified click handlers for better reactivity
    const handleFirstChoiceClick = (student) => {
      if (!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice)) {
        onStudentMove(student.id, student.first_choice);
      }
    };

    const handleSecondChoiceClick = (student) => {
      if (!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice)) {
        onStudentMove(student.id, student.second_choice);
      }
    };

    const handleThirdChoiceClick = (student) => {
      if (!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice)) {
        onStudentMove(student.id, student.third_choice);
      }
    };

    // Schedule modal functions
    const openScheduleModal = (childId, courseId, dayIndex, day, activity, event) => {
      activeScheduleModal.value = { childId, courseId, dayIndex, day, activity };
      
      // Calculate popover position
      const rect = event.target.getBoundingClientRect();
      popoverPosition.value = {
        x: rect.left + rect.width / 2,
        y: rect.top - 10 // Position above the element
      };
    };

    const closeScheduleModal = () => {
      activeScheduleModal.value = null;
      activePopoverForm.value = null;
    };

    const openPopoverForm = (childId, courseId, dayIndex) => {
      activePopoverForm.value = { childId, courseId, dayIndex };
    };

    const closePopoverForm = () => {
      activePopoverForm.value = null;
    };

    const getChildPreferences = (childId, dayIndex = null) => {
      const student = currentSemesterData.value?.children.find(s => s.id === childId);
      if (!student) return [];
      
      const currentDay = dayIndex !== null ? currentSemesterData.value?.schedule[dayIndex] : currentDayData.value;
      
      return [
        { choice: '1st', courseId: student.first_choice, name: getCourseName(student.first_choice, dayIndex) },
        { choice: '2nd', courseId: student.second_choice, name: getCourseName(student.second_choice, dayIndex) },
        { choice: '3rd', courseId: student.third_choice, name: getCourseName(student.third_choice, dayIndex) },
      ];
    };

    const getCurrentChoice = (childId, activity) => {
      const preferences = getChildPreferences(childId, activeScheduleModal.value?.dayIndex);
      return preferences.find(p => p.courseId === activity);
    };

    const getAvailableCoursesForMove = (dayIndex, excludeCourseId) => {
      const day = currentSemesterData.value?.schedule[dayIndex];
      if (!day) return [];
      
      const availableCourses = day.courses.filter(course => 
        course.id !== excludeCourseId && 
        getCourseAvailableSpotsForDay(dayIndex, course.id) > 0
      );
      
      // Add go-home option
      return [
        ...availableCourses.map(course => ({
          id: course.id,
          name: course.name,
          spotsAvailable: getCourseAvailableSpotsForDay(dayIndex, course.id)
        })),
        { id: 'go-home', name: 'Go Home', spotsAvailable: null }
      ];
    };

    const getMovementBlockingReason = (childId, targetCourseId, dayIndex) => {
      const child = currentSemesterData.value?.children.find(c => c.id === childId);
      if (!child) return 'Child not found';

      // Check if already in this course
      const currentEnrollment = child.enrollments[dayIndex];
      if (currentEnrollment === targetCourseId || (currentEnrollment === undefined && targetCourseId === 'go-home')) {
        return 'Already in this course/status';
      }

      if (targetCourseId === 'go-home') {
        return null; // Going home is always allowed
      }

      const day = currentSemesterData.value?.schedule[dayIndex];
      const course = day?.courses.find(c => c.id === targetCourseId);
      if (!course) return 'Course not found';

      // Check if course is forced full
      if (course.forcedFull) {
        return 'Course is marked as closed';
      }

      // Check grade eligibility
      const childGrade = child.class.charAt(0);
      if (!course.availableGrades.includes(childGrade)) {
        return `Not eligible (Grade ${childGrade} not allowed, only grades ${course.availableGrades.join(', ')})`;
      }

      // Check available spots
      const availableSpots = course.maxCapacity - course.enrolledChildren.length;
      if (availableSpots <= 0) {
        return `Course is full (${course.enrolledChildren.length}/${course.maxCapacity})`;
      }

      return null; // No blocking reason
    };

    const getCourseAvailableSpotsForDay = (dayIndex, courseId) => {
      if (courseId === 'go-home') return null;
      const day = currentSemesterData.value?.schedule[dayIndex];
      const course = day?.courses.find(c => c.id === courseId);
      if (!course) return 0;
      if (course.forcedFull) return 0;
      return course.maxCapacity - course.enrolledChildren.length;
    };

    const onResetToFirstChoices = () => {
      if (isEditing.value) return;
      console.log('Resetting all students to first choices');
      
      // Explicitly depend on selectedSemesterIndex to ensure reactivity
      const semesterIndex = selectedSemesterIndex.value;
      const dayIndex = selectedDayIndex.value;
      
      // Reset all students to their first choice for current day
      const semester = currentSemesterData.value;
      semester.children.forEach(student => {
        if (student.first_choice && student.first_choice !== 'go-home') {
          student.enrollments[dayIndex] = student.first_choice;
        } else {
          student.enrollments[dayIndex] = undefined;
        }
      });
      
      emit('trigger-event', { name: 'resetToFirstChoices' });
    };

    const onSemesterChange = async () => {
      if (isEditing.value) return;
      console.log('Semester changed to:', selectedSemesterIndex.value);
      
      // Set loading state during transition
      loading.value = true;
      
      selectedDayIndex.value = 0; // Reset to first day when semester changes
      currentSemester.value = semesters.value[selectedSemesterIndex.value].name;
      
      console.log('Semester changed to:', currentSemester.value, 'students:', currentSemesterData.value?.children?.length);
      
      // Force reactivity update
      await nextTick();
      
      // Force trigger reactivity on semesters ref
      triggerRef(semesters);
      
      // Clear loading state
      loading.value = false;
      
      emit('trigger-event', { name: 'semesterChange', event: { value: selectedSemesterIndex.value } });
    };

          // Initialize data when component mounts
    onMounted(async () => {
      console.log('Component mounted, initializing data...');
      initializeMockData();
      
      // Force reactivity update after initialization
      await nextTick();
      triggerRef(semesters);
      // Note: reactive objects don't need triggerRef
      
      // Set default visibility states (reactive objects are already initialized)
      console.log('Reactive objects initialized');
      
      console.log('Data initialized:', {
        semesters: semesters.value.length,
        currentSemester: currentSemester.value,
        selectedSemesterIndex: selectedSemesterIndex.value,
        selectedDayIndex: selectedDayIndex.value,
        loading: loading.value
      });
    });

    return {
      // State
      loading,
      searchQuery,
      selectedDayIndex,
      selectedSemesterIndex,
      currentSemester,
      isGoingHomeExpanded,
      isGoingHomeChildrenVisible,
      
      // Computed
      currentSemesterData,
      currentDayData,
      selectedDayName,
      filteredStudents,
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
      toggleCourseDetailsVisibility,
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
      autoEnrollStudents,
      handleFirstChoiceClick,
      handleSecondChoiceClick,
      handleThirdChoiceClick,
              openScheduleModal,
        closeScheduleModal,
        openPopoverForm,
        closePopoverForm,
        getChildPreferences,
        getCurrentChoice,
        getAvailableCoursesForMove,
        getMovementBlockingReason,
        getCourseAvailableSpotsForDay,
        activeScheduleModal,
        activePopoverForm,
        popoverPosition,
        
        // Notes functionality
        addNote,
        toggleNoteProblem,
        toggleProblemResolved,
        confirmDelete,
        deleteNote,
        startEditNote,
        cancelEditNote,
        saveEditNote,
        newNoteText,
        showAddNote,
        editingNote,
        editNoteText,
        formatTimestamp,
      
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
    border: none;
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

    &--locked {
      background-color: #f3f4f6;
      border-color: #d1d5db;
      color: #9ca3af;
      cursor: not-allowed;
    }
  }

  &__going-home-wait-btn {
    height: 24px;
    width: 24px;
    padding: 0;
    font-size: 12px;
    background-color: #f3f4f6;
    border: none;
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
    border: none;
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

    &--locked {
      background-color: #f3f4f6;
      border-color: #d1d5db;
      color: #9ca3af;
      cursor: not-allowed;
    }
  }

  &__waiting-list-wait-btn {
    height: 24px;
    width: 24px;
    padding: 0;
    font-size: 12px;
    background-color: #f3f4f6;
    border: none;
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
    gap: 6px;
    padding: 6px 12px;
    font-size: 13px;
    background-color: transparent;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s ease;
    color: #6b7280;

    &:hover {
      background-color: #f3f4f6;
      color: #374151;
    }

    &--locked {
      background-color: #dc2626;
      color: #ffffff;

      &:hover {
        background-color: #b91c1c;
      }
    }
  }

  &__course-lock-icon {
    width: 14px;
    height: 14px;
    stroke: currentColor;
    fill: none;
  }

  &__course-approval-btn {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    font-size: 13px;
    background-color: transparent;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s ease;
    color: #6b7280;

    &:hover {
      background-color: #f3f4f6;
      color: #374151;
    }

    &--approved {
      background-color: #dcfce7;
      color: #166534;

      &:hover {
        background-color: #bbf7d0;
      }
    }
  }

  &__course-approval-icon {
    width: 14px;
    height: 14px;
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
    width: 28px;
    height: 28px;
    padding: 0;
    font-size: 13px;
    background-color: transparent;
    border: none;
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

  &__course-notes-problem-icon {
    width: 8px;
    height: 8px;
    stroke: #dc2626;
    fill: none;
  }

  &__course-notes {
    margin-bottom: 8px;
  }

  &__course-notes-list {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-bottom: 8px;
  }

  &__course-note {
    padding: 8px;
    border-radius: 4px;
    font-size: 12px;
    border: 1px solid #e5e7eb;

    &--problem {
      background-color: #fef2f2;
      border-color: #fecaca;
    }

    &--resolved {
      background-color: #f0fdf4;
      border-color: #bbf7d0;
    }

    &--normal {
      background-color: #f9fafb;
      border-color: #e5e7eb;
    }
  }

  &__course-note-header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 8px;
    margin-bottom: 4px;
  }

  &__course-note-meta {
    display: flex;
    align-items: center;
    gap: 8px;
    flex: 1;
  }

  &__course-note-author {
    font-weight: 500;
    color: #374151;
  }

  &__course-note-timestamp {
    display: flex;
    align-items: center;
    gap: 2px;
    color: #6b7280;
    font-size: 11px;
  }

  &__course-note-clock-icon {
    width: 8px;
    height: 8px;
    stroke: currentColor;
    fill: none;
  }

  &__course-note-problem-badge {
    font-size: 10px;
    padding: 1px 4px;
    background-color: #dc2626;
    color: #ffffff;
    border-radius: 2px;
    font-weight: 500;
  }

  &__course-note-resolved-badge {
    font-size: 10px;
    padding: 1px 4px;
    background-color: #16a34a;
    color: #ffffff;
    border-radius: 2px;
    font-weight: 500;
  }

  &__course-note-actions {
    display: flex;
    gap: 4px;
  }

  &__course-note-action-btn {
    width: 24px;
    height: 24px;
    padding: 0;
    background-color: transparent;
    border: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 4px;
    transition: all 0.2s ease;

    &:hover {
      background-color: #f3f4f6;
    }

    &--problem {
      color: #dc2626;
      background-color: #fef2f2;

      &:hover {
        background-color: #fee2e2;
      }
    }

    &--resolve {
      color: #16a34a;

      &:hover {
        background-color: #f0fdf4;
      }
    }

    &--edit {
      color: #2563eb;

      &:hover {
        background-color: #eff6ff;
      }
    }

    &--delete {
      color: #dc2626;

      &:hover {
        background-color: #fef2f2;
      }
    }

    &--save {
      color: #16a34a;

      &:hover {
        background-color: #f0fdf4;
      }
    }

    &--cancel {
      color: #6b7280;

      &:hover {
        background-color: #f3f4f6;
      }
    }
  }

  &__course-note-action-icon {
    width: 14px;
    height: 14px;
    stroke: currentColor;
    fill: none;
  }

  &__course-note-edit {
    margin-bottom: 4px;
  }

  &__course-note-edit-form {
    display: flex;
    gap: 4px;
    align-items: center;
  }

  &__course-note-edit-input {
    flex: 1;
    padding: 4px 6px;
    border: 1px solid #d1d5db;
    border-radius: 2px;
    font-size: 12px;
    background-color: #ffffff;

    &:focus {
      outline: none;
      border-color: #2563eb;
      box-shadow: 0 0 0 1px rgba(37, 99, 235, 0.1);
    }
  }

  &__course-note-text {
    color: #374151;
    line-height: 1.4;
  }

  &__course-note-add {
    margin-top: 8px;
  }

  &__course-note-add-form {
    display: flex;
    gap: 4px;
    align-items: center;
  }

  &__course-note-add-input {
    flex: 1;
    padding: 4px 6px;
    border: 1px solid #d1d5db;
    border-radius: 2px;
    font-size: 12px;
    background-color: #ffffff;

    &:focus {
      outline: none;
      border-color: #2563eb;
      box-shadow: 0 0 0 1px rgba(37, 99, 235, 0.1);
    }

    &::placeholder {
      color: #9ca3af;
    }
  }

  &__course-note-add-btn {
    padding: 6px 12px;
    background-color: #2563eb;
    color: #ffffff;
    border: none;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: #1d4ed8;
    }
  }

  &__course-note-cancel-btn {
    padding: 6px 12px;
    background-color: transparent;
    color: #6b7280;
    border: none;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover {
      background-color: #f3f4f6;
      color: #374151;
    }
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
    width: 20px;
    height: 20px;
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
    width: 14px;
    height: 14px;
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
    border: none;
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

    &--locked {
      background-color: #f3f4f6;
      border-color: #d1d5db;
      color: #9ca3af;
      cursor: not-allowed;
    }
  }

  &__course-participants-wait-btn {
    height: 24px;
    width: 24px;
    padding: 0;
    font-size: 12px;
    background-color: #f3f4f6;
    border: none;
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

/* Modal styles */
.course-distribution__schedule-chip--clickable {
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

.course-distribution__popover-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1000;
  pointer-events: auto;
}

.course-distribution__popover {
  position: fixed;
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
  width: 256px;
  max-height: 80vh;
  overflow-y: auto;
  padding: 8px;
  pointer-events: auto;
  z-index: 1001;
}

.course-distribution__modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid #e5e7eb;
}

.course-distribution__modal-title {
  font-size: 16px;
  font-weight: 600;
  color: #111827;
  margin: 0;
}

.course-distribution__modal-close {
  background: none;
  border: none;
  cursor: pointer;
  padding: 4px;
  border-radius: 4px;
  color: #6b7280;
  transition: all 0.2s ease;

  &:hover {
    background-color: #f3f4f6;
    color: #374151;
  }
}

.course-distribution__modal-close-icon {
  width: 16px;
  height: 16px;
}

.course-distribution__modal-content {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.course-distribution__modal-day-title {
  font-size: 12px;
  margin: 0 0 8px 0;
  color: #374151;
}

.course-distribution__modal-current-choice {
  margin-bottom: 16px;
  padding: 8px 12px;
  background-color: #f0f9ff;
  border: 1px solid #bae6fd;
  border-radius: 6px;
}

.course-distribution__modal-current-text {
  font-size: 14px;
  color: #0369a1;
  margin: 0;
}

.course-distribution__modal-preferences {
  margin-bottom: 16px;
}

.course-distribution__modal-preferences-title {
  font-size: 14px;
  font-weight: 500;
  color: #6b7280;
  margin: 0 0 8px 0;
}

.course-distribution__modal-preferences-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.course-distribution__modal-preference-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 4px 8px;
  border-radius: 4px;
  transition: background-color 0.2s ease;

  &--active {
    background-color: #f0f9ff;
  }
}

.course-distribution__modal-preference-label {
  font-size: 12px;
  color: #374151;
}

.course-distribution__modal-preference-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  background-color: #f3f4f6;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  font-size: 12px;
  color: #374151;
  transition: all 0.2s ease;

  &--active {
    background-color: #3b82f6;
    border-color: #3b82f6;
    color: #ffffff;
  }

  &--clickable {
    cursor: pointer;

    &:hover {
      background-color: #e5e7eb;
      border-color: #9ca3af;
    }
  }
}

.course-distribution__modal-preference-capacity {
  font-size: 11px;
  color: #6b7280;
}

.course-distribution__modal-course-details {
  border-top: 1px solid #e5e7eb;
  padding-top: 16px;
}

.course-distribution__modal-course-title {
  font-size: 14px;
  font-weight: 500;
  color: #6b7280;
  margin: 0 0 12px 0;
}

.course-distribution__modal-enrolled-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.course-distribution__modal-enrolled-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 6px 8px;
  background-color: #f9fafb;
  border-radius: 4px;
}

.course-distribution__modal-enrolled-name {
  font-size: 12px;
  color: #374151;
}

.course-distribution__modal-move-select {
  padding: 2px 6px;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 11px;
  background-color: #ffffff;
  color: #374151;
  cursor: pointer;

  &:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.1);
  }
}

.course-distribution__modal-no-students {
  font-size: 12px;
  color: #6b7280;
  text-align: center;
  padding: 8px;
}

.course-distribution__modal-actions {
  margin-top: 16px;
}

.course-distribution__modal-action-btn {
  width: 100%;
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;

  &--default {
    background-color: #3b82f6;
    color: #ffffff;

    &:hover {
      background-color: #2563eb;
    }
  }

  &--destructive {
    background-color: #ef4444;
    color: #ffffff;

    &:hover {
      background-color: #dc2626;
    }
  }
}

/* Inline Form Styles (matching React's renderPopoverInlineForm) */
.course-distribution__modal-inline-form {
  margin-top: 8px;
  padding: 8px;
  background-color: rgba(0, 0, 0, 0.05);
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  font-size: 12px;
}

.course-distribution__modal-inline-form-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.course-distribution__modal-inline-form-title {
  display: flex;
  align-items: center;
  gap: 4px;
}

.course-distribution__modal-inline-form-course-name {
  font-size: 12px;
  color: #374151;
}

.course-distribution__modal-inline-form-badge {
  display: inline-block;
  padding: 1px 4px;
  background-color: transparent;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 11px;
  color: #6b7280;
}

.course-distribution__modal-inline-form-close {
  background: none;
  border: none;
  cursor: pointer;
  padding: 2px;
  border-radius: 2px;
  color: #6b7280;
  transition: all 0.2s ease;

  &:hover {
    background-color: #f3f4f6;
    color: #374151;
  }
}

.course-distribution__modal-inline-form-close-icon {
  width: 8px;
  height: 8px;
}

.course-distribution__modal-inline-form-enrolled {
  margin-bottom: 8px;
}

.course-distribution__modal-inline-form-enrolled-title {
  font-size: 12px;
  color: #6b7280;
  margin: 0 0 4px 0;
}

.course-distribution__modal-inline-form-enrolled-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.course-distribution__modal-inline-form-enrolled-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 2px 4px;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 2px;
}

.course-distribution__modal-inline-form-enrolled-name {
  font-size: 12px;
  color: #374151;
}

.course-distribution__modal-inline-form-move-select {
  height: 16px;
  width: 64px;
  font-size: 12px;
  padding: 0 4px;
  border: 1px solid #d1d5db;
  border-radius: 2px;
  background-color: #ffffff;
  color: #374151;
  cursor: pointer;

  &:focus {
    outline: none;
    border-color: #3b82f6;
  }
}

.course-distribution__modal-inline-form-no-students {
  margin-bottom: 8px;
  padding: 4px;
  text-align: center;
}

.course-distribution__modal-inline-form-no-students-text {
  font-size: 12px;
  color: #6b7280;
  margin: 0;
}

.course-distribution__modal-inline-form-action-btn {
  height: 20px;
  padding: 0 8px;
  font-size: 12px;
  width: 100%;
  border: none;
  border-radius: 4px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;

  &--default {
    background-color: #3b82f6;
    color: #ffffff;

    &:hover {
      background-color: #2563eb;
    }
  }

  &--destructive {
    background-color: #ef4444;
    color: #ffffff;

    &:hover {
      background-color: #dc2626;
    }
  }
}
</style>
