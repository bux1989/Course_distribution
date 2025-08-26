<template>
  <div class="course-distribution">
    <!-- Modern Header Section -->
    <div class="course-distribution__header">
      <div class="course-distribution__header-left">
        <h1 class="course-distribution__title">Course Enrollment Distribution System</h1>
        <div class="course-distribution__subtitle-container">
          <span class="course-distribution__semester">{{ currentSemester || 'Fall 2024' }}</span>
          <span class="course-distribution__separator">•</span>
          <span class="course-distribution__day">{{ selectedDayName }}</span>
        </div>
      </div>
      
      <div class="course-distribution__header-right">
        <button 
          class="course-distribution__reset-btn"
          @click="onResetToFirstChoices"
          title="Reset all students to their first choice (Ctrl+R)"
        >
          <svg class="course-distribution__reset-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"/>
            <path d="M21 3v5h-5"/>
            <path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"/>
            <path d="M3 21v-5h5"/>
          </svg>
          Reset
        </button>
        
        <button 
          class="course-distribution__help-btn"
          @click="toggleKeyboardHelp"
          title="Keyboard shortcuts (Ctrl+H)"
        >
          <svg class="course-distribution__help-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="10"/>
            <path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"/>
            <path d="M12 17h.01"/>
          </svg>
          Help
        </button>
        
        <button 
          class="course-distribution__undo-btn"
          @click="undo"
          :disabled="!canUndo"
          title="Undo last action (Ctrl+Z)"
        >
          <svg class="course-distribution__undo-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M3 7v6h6"/>
            <path d="M21 17a9 9 0 0 0-9-9 9 9 0 0 0-6 2.3L3 13"/>
          </svg>
          Undo
        </button>
        
        <button 
          class="course-distribution__redo-btn"
          @click="redo"
          :disabled="!canRedo"
          title="Redo last undone action (Ctrl+Y)"
        >
          <svg class="course-distribution__redo-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 7v6h-6"/>
            <path d="M3 17a9 9 0 0 1 9-9 9 9 0 0 1 6 2.3l3-2.7"/>
          </svg>
          Redo
        </button>
        
        <div class="course-distribution__search">
          <svg class="course-distribution__search-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="11" cy="11" r="8"/>
            <path d="m21 21-4.35-4.35"/>
          </svg>
          <input
            type="text"
            placeholder="Search students or courses..."
            v-model="searchQuery"
            @input="onSearchChange($event.target.value)"
            class="course-distribution__search-input"
          />
        </div>
        
        <button 
          class="course-distribution__filters-btn"
          @click="showAdvancedFilters = !showAdvancedFilters"
          :class="{ 'active': showAdvancedFilters }"
          title="Advanced filters"
        >
          <svg class="course-distribution__filters-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <line x1="21" y1="6" x2="3" y2="6"/>
            <line x1="21" y1="12" x2="3" y2="12"/>
            <line x1="21" y1="18" x2="3" y2="18"/>
            <circle cx="4" cy="6" r="1"/>
            <circle cx="4" cy="12" r="1"/>
            <circle cx="4" cy="18" r="1"/>
          </svg>
          Filters
        </button>
        
        <div class="course-distribution__select-wrapper">
          <select
            :value="selectedSemesterIndex"
            @change="onSemesterChange($event.target.value)"
            class="course-distribution__semester-select"
          >
            <option value="0">Fall 2024</option>
            <option value="1">Spring 2025</option>
            <option value="2">Summer 2025</option>
          </select>
        </div>
        
        <div class="course-distribution__select-wrapper">
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

    <!-- Advanced Filters Panel -->
    <div v-if="showAdvancedFilters" class="course-distribution__filters-panel">
      <div class="course-distribution__filters-header">
        <h3 class="course-distribution__filters-title">Advanced Filters</h3>
        <button @click="clearAllFilters" class="course-distribution__clear-filters-btn">Clear All</button>
      </div>
      
      <div class="course-distribution__filters-content">
        <div class="course-distribution__filter-group">
          <label class="course-distribution__filter-label">Student Status</label>
          <select v-model="filterOptions.status" class="course-distribution__filter-select">
            <option value="all">All Students</option>
            <option value="enrolled">Enrolled</option>
            <option value="waiting">Waiting List</option>
            <option value="going-home">Going Home</option>
          </select>
        </div>
        
        <div class="course-distribution__filter-group">
          <label class="course-distribution__filter-label">Course</label>
          <select v-model="filterOptions.course" class="course-distribution__filter-select">
            <option value="all">All Courses</option>
            <option v-for="course in allCourses" :key="course.id" :value="course.id">
              {{ course.name }}
            </option>
          </select>
        </div>
        
        <div class="course-distribution__filter-group">
          <label class="course-distribution__filter-label">Preference Match</label>
          <select v-model="filterOptions.preference" class="course-distribution__filter-select">
            <option value="all">All Preferences</option>
            <option value="first">First Choice</option>
            <option value="second">Second Choice</option>
            <option value="third">Third Choice</option>
          </select>
        </div>
        
        <div class="course-distribution__filter-group">
          <label class="course-distribution__filter-label">Course Capacity</label>
          <select v-model="filterOptions.capacity" class="course-distribution__filter-select">
            <option value="all">All Capacities</option>
            <option value="available">Available</option>
            <option value="full">Full</option>
            <option value="overfilled">Overfilled</option>
          </select>
        </div>
        
        <div class="course-distribution__filter-group">
          <label class="course-distribution__filter-label">Approval Status</label>
          <select v-model="filterOptions.approved" class="course-distribution__filter-select">
            <option value="all">All Courses</option>
            <option value="approved">Approved</option>
            <option value="not-approved">Not Approved</option>
          </select>
        </div>
      </div>
      
      <div class="course-distribution__filters-summary">
        <span class="course-distribution__filters-text">{{ getFilterSummary() }}</span>
      </div>
    </div>

    <div v-if="loading" class="course-distribution__loading">Loading...</div>

    <div v-else-if="filteredCourses.length === 0 && totalEnrolledForDay === 0" class="course-distribution__empty">
      <p>No courses or students available for the selected criteria.</p>
    </div>

    <div v-else class="course-distribution__content">
      <!-- Enhanced Problems Section -->
      <div v-if="unresolvedProblems.length > 0" class="course-distribution__problems">
        <div class="course-distribution__problems-header">
          <div class="course-distribution__problems-title">
            <svg class="course-distribution__problems-icon" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.732-.833-2.464 0L4.35 16.5c-.77.833.192 2.5 1.732 2.5z" />
            </svg>
            <h3 class="course-distribution__problems-heading">Problems Requiring Resolution</h3>
            <span class="course-distribution__problems-count">{{ unresolvedProblems.length }}</span>
          </div>
          <p class="course-distribution__problems-description">
            Issues that need attention before course distribution can be finalized
          </p>
        </div>
        
        <div class="course-distribution__problems-list">
          <div 
            v-for="note in unresolvedProblems" 
            :key="note.id" 
            class="course-distribution__problem-item"
          >
            <div class="course-distribution__problem-content">
              <div class="course-distribution__problem-header">
                <div class="course-distribution__problem-meta">
                  <span class="course-distribution__problem-course">{{ getCourseName(note.course_id) }}</span>
                  <span class="course-distribution__problem-author">{{ note.author }}</span>
                  <span class="course-distribution__problem-date">{{ formatDate(note.created_at) }}</span>
                </div>
                <div class="course-distribution__problem-actions">
                  <button
                    @click="onResolveNote(note.id)"
                    class="course-distribution__resolve-btn"
                    title="Mark as resolved"
                  >
                    <svg class="course-distribution__resolve-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
                    </svg>
                    Resolve
                  </button>
                </div>
              </div>
              <p class="course-distribution__problem-text">{{ note.text }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Waiting List -->
      <div v-if="waitingListStudents.length > 0" class="cd-card">
        <div class="cd-card__top">
          <h4 class="cd-card__title">
            <span>Waiting List</span>
            <span class="cd-capacity cd-capacity--neutral">{{ waitingListStudents.length }}</span>
          </h4>
        </div>
        <div class="cd-card__underline"></div>

        <div class="cd-participants">
          <!-- Header row aligned with columns -->
          <div class="cd-participants__bar">
            <div class="cd-grid cd-grid--header">
              <div class="cd-col cd-col--participants">Participants</div>
              <div class="cd-col">1. wish</div>
              <div class="cd-col">2. wish</div>
              <div class="cd-col">3. wish</div>
              <div class="cd-col cd-col--center">Wait</div>
            </div>
          </div>

          <!-- Rows -->
          <div class="cd-rows">
            <div v-for="student in waitingListStudents" :key="student.id + '-' + selectedDayNumber" class="cd-row"
              :class="{ 'dragging': isDragging && draggedStudent?.id === student.id }"
              draggable="true"
              @dragstart="onDragStart($event, student)"
              @dragend="onDragEnd"
            >
              <div class="cd-grid">
                <div class="cd-col cd-col--participants cd-col--person">
                  <div>
                    <span 
                      class="cd-name cd-name--clickable" 
                      @click="toggleStudentPopover(student.id, $event)"
                      :title="`Click to view ${student.name}'s details`"
                    >
                      {{ student.name }}
                    </span>
                    <div class="cd-schedule-info">
                      <span 
                        class="cd-schedule-chip cd-schedule-chip--clickable"
                        @click="toggleSchedulePopover(student.id, { day: 'Tuesday', period: 1, activity: 'football' }, $event)"
                        :title="`Click to view Tuesday schedule`"
                      >
                        Tue (1): football
                      </span>
                      <span 
                        class="cd-schedule-chip cd-schedule-chip--clickable"
                        @click="toggleSchedulePopover(student.id, { day: 'Wednesday', period: 2, activity: 'volleyball' }, $event)"
                        :title="`Click to view Wednesday schedule`"
                      >
                        Wed (2): volleyball
                      </span>
                    </div>
                  </div>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    v-if="student.first_choice"
                    class="cd-pref-btn"
                    :class="[
                      isCurrentChoice(student, 1) ? 'cd-pref-btn--first cd-pref-btn--current' : 'cd-pref-btn--first',
                      isLockedTarget(student.first_choice) ? 'cd-pref-btn--locked' : ''
                    ]"
                    :disabled="isCurrentChoice(student, 1) || isLockedTarget(student.first_choice)"
                    @click="!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice) && openMovementPopover(student.id, 'first', $event)"
                    :title="isLockedTarget(student.first_choice) ? 'Course locked' : formatPreferenceDisplay(student.first_choice)"
                  >
                    {{ formatPreferenceDisplay(student.first_choice) }}
                  </button>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    v-if="student.second_choice"
                    class="cd-pref-btn"
                    :class="[
                      isCurrentChoice(student, 2) ? 'cd-pref-btn--second cd-pref-btn--current' : 'cd-pref-btn--second',
                      isLockedTarget(student.second_choice) ? 'cd-pref-btn--locked' : ''
                    ]"
                    :disabled="isCurrentChoice(student, 2) || isLockedTarget(student.second_choice)"
                    @click="!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice) && openMovementPopover(student.id, 'second', $event)"
                    :title="isLockedTarget(student.second_choice) ? 'Course locked' : formatPreferenceDisplay(student.second_choice)"
                  >
                    {{ formatPreferenceDisplay(student.second_choice) }}
                  </button>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    v-if="student.third_choice"
                    class="cd-pref-btn"
                    :class="[
                      isCurrentChoice(student, 3) ? 'cd-pref-btn--third cd-pref-btn--current' : 'cd-pref-btn--third',
                      isLockedTarget(student.third_choice) ? 'cd-pref-btn--locked' : ''
                    ]"
                    :disabled="isCurrentChoice(student, 3) || isLockedTarget(student.third_choice)"
                    @click="!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice) && openMovementPopover(student.id, 'third', $event)"
                    :title="isLockedTarget(student.third_choice) ? 'Course locked' : formatPreferenceDisplay(student.third_choice)"
                  >
                    {{ formatPreferenceDisplay(student.third_choice) }}
                  </button>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    class="cd-wait-btn"
                    title="Move to waiting list"
                    @click="onStudentMove(student.id, 'waiting')"
                    disabled
                  >
                    W
                  </button>
                </div>
              </div>
            </div>

            <p v-if="waitingListStudents.length === 0" class="cd-empty-text">No students waiting</p>
          </div>
        </div>
      </div>

      <!-- Going Home -->
      <div class="cd-card">
        <div class="cd-card__top">
          <h4 class="cd-card__title">
            <span>Going Home</span>
            <span class="cd-capacity cd-capacity--neutral">{{ goingHomeStudents.length }}</span>
          </h4>
        </div>
        <div class="cd-card__underline"></div>

        <div class="cd-participants">
          <!-- Header row aligned with columns -->
          <div class="cd-participants__bar">
            <div class="cd-grid cd-grid--header">
              <div class="cd-col cd-col--participants">Participants</div>
              <div class="cd-col">1. wish</div>
              <div class="cd-col">2. wish</div>
              <div class="cd-col">3. wish</div>
              <div class="cd-col cd-col--center">Wait</div>
            </div>
          </div>

          <!-- Rows -->
          <div class="cd-rows">
            <div v-for="student in goingHomeStudents" :key="student.id + '-' + selectedDayNumber" class="cd-row"
              :class="{ 'dragging': isDragging && draggedStudent?.id === student.id }"
              draggable="true"
              @dragstart="onDragStart($event, student)"
              @dragend="onDragEnd"
            >
              <div class="cd-grid">
                <div class="cd-col cd-col--participants cd-col--person">
                  <div>
                    <span 
                      class="cd-name cd-name--clickable" 
                      @click="toggleStudentPopover(student.id, $event)"
                      :title="`Click to view ${student.name}'s details`"
                    >
                      {{ student.name }}
                    </span>
                    <div class="cd-schedule-info">
                      <span 
                        class="cd-schedule-chip cd-schedule-chip--clickable"
                        @click="toggleSchedulePopover(student.id, { day: 'Tuesday', period: 2, activity: 'archery' }, $event)"
                        :title="`Click to view Tuesday schedule`"
                      >
                        Tue (2): archery
                      </span>
                      <span 
                        class="cd-schedule-chip cd-schedule-chip--clickable"
                        @click="toggleSchedulePopover(student.id, { day: 'Wednesday', period: 3, activity: 'volleyball' }, $event)"
                        :title="`Click to view Wednesday schedule`"
                      >
                        Wed (3): volleyball
                      </span>
                    </div>
                  </div>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    v-if="student.first_choice"
                    class="cd-pref-btn"
                    :class="[
                      isCurrentChoice(student, 1) ? 'cd-pref-btn--first cd-pref-btn--current' : 'cd-pref-btn--first',
                      isLockedTarget(student.first_choice) ? 'cd-pref-btn--locked' : ''
                    ]"
                    :disabled="isCurrentChoice(student, 1) || isLockedTarget(student.first_choice)"
                    @click="!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice) && onStudentMove(student.id, student.first_choice)"
                    :title="isLockedTarget(student.first_choice) ? 'Course locked' : formatPreferenceDisplay(student.first_choice)"
                  >
                    {{ formatPreferenceDisplay(student.first_choice) }}
                  </button>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    v-if="student.second_choice"
                    class="cd-pref-btn"
                    :class="[
                      isCurrentChoice(student, 2) ? 'cd-pref-btn--second cd-pref-btn--current' : 'cd-pref-btn--second',
                      isLockedTarget(student.second_choice) ? 'cd-pref-btn--locked' : ''
                    ]"
                    :disabled="isCurrentChoice(student, 2) || isLockedTarget(student.second_choice)"
                    @click="!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice) && onStudentMove(student.id, student.second_choice)"
                    :title="isLockedTarget(student.second_choice) ? 'Course locked' : formatPreferenceDisplay(student.second_choice)"
                  >
                    {{ formatPreferenceDisplay(student.second_choice) }}
                  </button>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    v-if="student.third_choice"
                    class="cd-pref-btn"
                    :class="[
                      isCurrentChoice(student, 3) ? 'cd-pref-btn--third cd-pref-btn--current' : 'cd-pref-btn--third',
                      isLockedTarget(student.third_choice) ? 'cd-pref-btn--locked' : ''
                    ]"
                    :disabled="isCurrentChoice(student, 3) || isLockedTarget(student.third_choice)"
                    @click="!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice) && onStudentMove(student.id, student.third_choice)"
                    :title="isLockedTarget(student.third_choice) ? 'Course locked' : formatPreferenceDisplay(student.third_choice)"
                  >
                    {{ formatPreferenceDisplay(student.third_choice) }}
                  </button>
                </div>

                <div class="cd-col cd-col--center">
                  <button
                    class="cd-wait-btn"
                    title="Move to waiting list"
                    @click="onStudentMove(student.id, 'waiting')"
                  >
                    W
                  </button>
                </div>
              </div>
            </div>

            <p v-if="goingHomeStudents.length === 0" class="cd-empty-text">No students</p>
          </div>
        </div>
      </div>

      <!-- Modern Course Sections -->
      <div v-for="course in filteredCourses" :key="course.id" class="course-distribution__section" :class="[
        getCourseSectionClass(course),
        {
          'drag-over': dragOverCourse === course.id && !course.is_locked && (enrolledByCourseId[course.id]?.length || 0) < course.max_capacity,
          'drag-over-full': dragOverCourse === course.id && (enrolledByCourseId[course.id]?.length || 0) >= course.max_capacity,
          'drag-over-locked': dragOverCourse === course.id && course.is_locked
        }
      ]"
        @dragover="onDragOver($event, course)"
        @dragleave="onDragLeave"
        @drop="onDrop($event, course)"
      >
        <div class="course-distribution__section-header">
          <div class="course-distribution__section-title">
            <button 
              class="course-distribution__course-name"
              @click="toggleCourseDetails(course.id)"
              :title="course.name"
            >
              {{ course.name }}
            </button>
            
            <div class="course-distribution__section-badges">
              <span class="course-distribution__capacity-badge" :class="getCapacityBadgeClass(course)">
                {{ getCourseCapacity(course) }}
              </span>
              <span class="course-distribution__status-badge" :class="getStatusBadgeClass(course)">
                {{ statusBadgeText(course) }}
              </span>
            </div>
          </div>
          
          <div class="course-distribution__section-actions">
            <button
              class="course-distribution__action-btn course-distribution__action-btn--lock"
              @click="onLockToggle(course.id, !course.is_locked)"
              :title="course.is_locked ? 'Unlock course' : 'Lock course'"
            >
              <svg v-if="course.is_locked" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect width="18" height="11" x="3" y="11" rx="2" ry="2"/>
                <circle cx="12" cy="16" r="1"/>
                <path d="M7 11V7a5 5 0 0 1 10 0v4"/>
              </svg>
              <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect width="18" height="11" x="3" y="11" rx="2" ry="2"/>
                <circle cx="12" cy="16" r="1"/>
                <path d="M7 11V7a5 5 0 0 1 9.9-.9"/>
              </svg>
            </button>
            
            <button
              class="course-distribution__action-btn course-distribution__action-btn--approve"
              @click="onApproveClick(course.id)"
              :title="isCourseApproved(course.id) ? 'Course approved' : 'Approve course'"
              :disabled="isCourseApproved(course.id)"
            >
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7"/>
              </svg>
            </button>
            
            <button
              class="course-distribution__action-btn course-distribution__action-btn--notes"
              @click="onNotesClick(course.id)"
              title="View notes"
            >
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/>
                <polyline points="14,2 14,8 20,8"/>
                <line x1="16" y1="13" x2="8" y2="13"/>
                <line x1="16" y1="17" x2="8" y2="17"/>
                <polyline points="10,9 9,9 8,9"/>
              </svg>
              <span v-if="getCourseNotesCount(course.id).total" class="course-distribution__notes-badge">
                {{ getCourseNotesCount(course.id).unresolved || getCourseNotesCount(course.id).total }}
              </span>
            </button>
            
            <button
              class="course-distribution__action-btn course-distribution__action-btn--expand"
              @click="toggleParticipants(course.id)"
              :title="isParticipantsVisible(course.id) ? 'Hide participants' : 'Show participants'"
            >
              <svg v-if="isParticipantsVisible(course.id)" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="m18 15-6-6-6 6"/>
              </svg>
              <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="m6 9 6 6 6-6"/>
              </svg>
            </button>
          </div>
        </div>

        <!-- Optional meta line -->
        <div class="cd-meta">
          <span v-if="course.teacher">Teacher: {{ course.teacher }}</span>
          <span v-if="course.room">Room: {{ course.room }}</span>
          <span v-if="course.day">Day: {{ course.day }}</span>
        </div>

        <div class="cd-participants">
          <!-- Header row aligned with columns + chevron -->
          <div class="cd-participants__bar">
            <div class="cd-grid cd-grid--header">
              <div class="cd-col cd-col--participants">Participants</div>
              <div class="cd-col">1. wish</div>
              <div class="cd-col">2. wish</div>
              <div class="cd-col">3. wish</div>
              <div class="cd-col cd-col--center">Wait</div>
            </div>
            <button
              class="cd-icon-btn cd-icon-btn--chev"
              :title="isParticipantsVisible(course.id) ? 'Hide participants' : 'Show participants'"
              @click="toggleParticipants(course.id)"
            >
              <svg v-if="isParticipantsVisible(course.id)" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="cd-icon cd-icon--blue">
                <path d="m18 15-6-6-6 6"></path>
              </svg>
              <svg v-else xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="cd-icon cd-icon--blue">
                <path d="m6 9 6 6 6-6"></path>
              </svg>
            </button>
          </div>

          <template v-if="isParticipantsVisible(course.id)">
            <div class="cd-participants__content">
              <div v-for="student in (enrolledByCourseId[course.id] || [])" :key="student.id + '-' + selectedDayNumber"
                class="cd-row"
                :class="{ 'dragging': isDragging && draggedStudent?.id === student.id }"
                draggable="true"
                @dragstart="onDragStart($event, student)"
                @dragend="onDragEnd"
              >
                <div class="cd-grid">
                  <div class="cd-col cd-col--participants cd-col--person">
                    <div>
                      <span 
                        class="cd-name cd-name--clickable" 
                        @click="toggleStudentPopover(student.id, $event)"
                        :title="`Click to view ${student.name}'s details`"
                      >
                        {{ student.name }}
                      </span>
                      <div class="cd-schedule-info">
                        <span 
                          class="cd-schedule-chip cd-schedule-chip--clickable"
                          @click="toggleSchedulePopover(student.id, { day: 'Tuesday', period: 1, activity: 'football' }, $event)"
                          :title="`Click to view Tuesday schedule`"
                        >
                          Tue (1): football
                        </span>
                        <span 
                          class="cd-schedule-chip cd-schedule-chip--clickable"
                          @click="toggleSchedulePopover(student.id, { day: 'Wednesday', period: 2, activity: 'football' }, $event)"
                          :title="`Click to view Wednesday schedule`"
                        >
                          Wed (2): football
                        </span>
                      </div>
                    </div>
                  </div>

                  <div class="cd-col cd-col--center">
                    <button
                      v-if="student.first_choice"
                      class="cd-pref-btn"
                      :class="[
                        isCurrentChoice(student, 1) ? 'cd-pref-btn--first cd-pref-btn--current' : 'cd-pref-btn--first',
                        isLockedTarget(student.first_choice) ? 'cd-pref-btn--locked' : ''
                      ]"
                      :disabled="isCurrentChoice(student, 1) || isLockedTarget(student.first_choice)"
                      @click="!isCurrentChoice(student, 1) && !isLockedTarget(student.first_choice) && openMovementPopover(student.id, 'first', $event)"
                      :title="isLockedTarget(student.first_choice) ? 'Course locked' : formatPreferenceDisplay(student.first_choice)"
                    >
                      {{ formatPreferenceDisplay(student.first_choice) }}
                    </button>
                  </div>

                  <div class="cd-col cd-col--center">
                    <button
                      v-if="student.second_choice"
                      class="cd-pref-btn"
                      :class="[
                        isCurrentChoice(student, 2) ? 'cd-pref-btn--second cd-pref-btn--current' : 'cd-pref-btn--second',
                        isLockedTarget(student.second_choice) ? 'cd-pref-btn--locked' : ''
                      ]"
                      :disabled="isCurrentChoice(student, 2) || isLockedTarget(student.second_choice)"
                      @click="!isCurrentChoice(student, 2) && !isLockedTarget(student.second_choice) && openMovementPopover(student.id, 'second', $event)"
                      :title="isLockedTarget(student.second_choice) ? 'Course locked' : formatPreferenceDisplay(student.second_choice)"
                    >
                      {{ formatPreferenceDisplay(student.second_choice) }}
                    </button>
                  </div>

                  <div class="cd-col cd-col--center">
                    <button
                      v-if="student.third_choice"
                      class="cd-pref-btn"
                      :class="[
                        isCurrentChoice(student, 3) ? 'cd-pref-btn--third cd-pref-btn--current' : 'cd-pref-btn--third',
                        isLockedTarget(student.third_choice) ? 'cd-pref-btn--locked' : ''
                      ]"
                      :disabled="isCurrentChoice(student, 3) || isLockedTarget(student.third_choice)"
                      @click="!isCurrentChoice(student, 3) && !isLockedTarget(student.third_choice) && openMovementPopover(student.id, 'third', $event)"
                      :title="isLockedTarget(student.third_choice) ? 'Course locked' : formatPreferenceDisplay(student.third_choice)"
                    >
                      {{ formatPreferenceDisplay(student.third_choice) }}
                    </button>
                  </div>

                  <div class="cd-col cd-col--center">
                    <button
                      class="cd-wait-btn"
                      title="Move to waiting list"
                      @click="onStudentMove(student.id, 'waiting')"
                    >
                      W
                    </button>
                  </div>
                </div>
              </div>

              <p v-if="(enrolledByCourseId[course.id] || []).length === 0" class="cd-empty-text">No students enrolled</p>
            </div>
          </template>
        </div>
      </div>
    </div>
  </div>

  <!-- Popover Component -->
  <div 
    v-if="activePopover" 
    class="course-distribution__popover"
    :style="{
      left: popoverPosition.x + 'px',
      top: popoverPosition.y + 'px'
    }"
    @click.stop
  >
    <!-- Student Information Popover -->
    <div v-if="activePopover === 'student' && popoverContent" class="course-distribution__popover-content">
      <div class="course-distribution__popover-header">
        <h4 class="course-distribution__popover-title">{{ popoverContent.name }}</h4>
        <button 
          class="course-distribution__popover-close"
          @click="hidePopover"
          title="Close"
        >
          ×
        </button>
      </div>
      
      <div class="course-distribution__popover-body">
        <div class="course-distribution__popover-section">
          <h5 class="course-distribution__popover-section-title">Current Enrollment</h5>
          <p class="course-distribution__popover-text">
            {{ popoverContent.current_enrollment ? getCourseName(popoverContent.current_enrollment) : 'Not enrolled' }}
          </p>
        </div>
        
        <div class="course-distribution__popover-section">
          <h5 class="course-distribution__popover-section-title">Preferences</h5>
          <div class="course-distribution__popover-preferences">
            <div class="course-distribution__popover-preference">
              <span class="course-distribution__popover-preference-label">1st:</span>
              <span class="course-distribution__popover-preference-value">{{ formatPreferenceDisplay(popoverContent.first_choice) }}</span>
            </div>
            <div class="course-distribution__popover-preference">
              <span class="course-distribution__popover-preference-label">2nd:</span>
              <span class="course-distribution__popover-preference-value">{{ formatPreferenceDisplay(popoverContent.second_choice) }}</span>
            </div>
            <div class="course-distribution__popover-preference">
              <span class="course-distribution__popover-preference-label">3rd:</span>
              <span class="course-distribution__popover-preference-value">{{ formatPreferenceDisplay(popoverContent.third_choice) }}</span>
            </div>
          </div>
        </div>
        
        <div class="course-distribution__popover-section">
          <h5 class="course-distribution__popover-section-title">Weekly Schedule</h5>
          <div class="course-distribution__popover-schedule">
            <div 
              v-for="(schedule, index) in getStudentSchedule(popoverContent)" 
              :key="index"
              class="course-distribution__popover-schedule-item"
            >
              <span class="course-distribution__popover-schedule-day">{{ schedule.day }}</span>
              <span class="course-distribution__popover-schedule-period">({{ schedule.period }})</span>
              <span class="course-distribution__popover-schedule-activity">{{ schedule.activity }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Schedule Information Popover -->
    <div v-if="activePopover === 'schedule' && popoverContent" class="course-distribution__popover-content">
      <div class="course-distribution__popover-header">
        <h4 class="course-distribution__popover-title">{{ popoverContent.student.name }} - Schedule</h4>
        <button 
          class="course-distribution__popover-close"
          @click="hidePopover"
          title="Close"
        >
          ×
        </button>
      </div>
      
      <div class="course-distribution__popover-body">
        <div class="course-distribution__popover-section">
          <h5 class="course-distribution__popover-section-title">Full Weekly Schedule</h5>
          <div class="course-distribution__popover-schedule">
            <div 
              v-for="(schedule, index) in popoverContent.fullSchedule" 
              :key="index"
              class="course-distribution__popover-schedule-item"
              :class="{ 'course-distribution__popover-schedule-item--highlighted': schedule.day === popoverContent.schedule.day }"
            >
              <span class="course-distribution__popover-schedule-day">{{ schedule.day }}</span>
              <span class="course-distribution__popover-schedule-period">({{ schedule.period }})</span>
              <span class="course-distribution__popover-schedule-activity">{{ schedule.activity }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Notes Modal -->
  <div v-if="isNotesOpen" class="notes-modal" @click.self="closeNotes">
    <div class="notes-modal__content">
      <div class="notes-modal__header">
        <h3 class="notes-modal__title">Course Notes</h3>
        <div class="notes-modal__meta" v-if="activeCourseForNotes">
          <span class="notes-modal__count">Total: {{ notesCounts.total }}</span>
          <span class="notes-modal__count notes-modal__count--warn">Unresolved: {{ notesCounts.unresolved }}</span>
          <span class="notes-modal__count notes-modal__count--ok">Resolved: {{ notesCounts.resolved }}</span>
        </div>
        <button class="notes-modal__close" @click="closeNotes" title="Close">×</button>
      </div>

      <div class="notes-modal__list" v-if="getNotesForActiveCourse.length">
        <div v-for="n in getNotesForActiveCourse" :key="n.id" class="notes-item" :class="{ 'notes-item--resolved': n.is_resolved }">
          <div class="notes-item__top">
            <span class="notes-item__course">{{ getCourseName(n.course_id) }}</span>
            <span class="notes-item__author">{{ n.author }}</span>
            <span class="notes-item__date">{{ formatDate(n.created_at) }}</span>
          </div>
          <p class="notes-item__text">{{ n.text }}</p>
          <div class="notes-item__actions">
            <button class="notes-item__btn" @click="toggleNoteResolved(n.id)">
              {{ n.is_resolved ? 'Mark Unresolved' : 'Mark Resolved' }}
            </button>
          </div>
        </div>
      </div>
      <p v-else class="notes-modal__empty">No notes for this course.</p>

      <div class="notes-modal__add">
        <input class="notes-modal__input" v-model="newNoteText" placeholder="Add a new note..." @keyup.enter="addNoteForCourse" />
        <button class="notes-modal__add-btn" @click="addNoteForCourse">Add</button>
      </div>
    </div>
  </div>

  <!-- Movement Popover -->
  <div 
    v-if="isMovementPopoverOpen" 
    class="course-distribution__movement-popover"
    :style="{
      left: movementPopoverPosition.x + 'px',
      top: movementPopoverPosition.y + 'px'
    }"
    @click.stop
  >
    <div class="course-distribution__movement-content">
      <div class="course-distribution__movement-header">
        <h4 class="course-distribution__movement-title">{{ movementPopoverData.student.name }}</h4>
        <button 
          class="course-distribution__movement-close"
          @click="closeMovementPopover"
          title="Close"
        >
          ×
        </button>
      </div>
      
      <div class="course-distribution__movement-body">
        <div class="course-distribution__movement-section">
          <h5 class="course-distribution__movement-section-title">Available Courses</h5>
          <div class="course-distribution__movement-courses">
            <div 
              v-for="course in movementPopoverData.availableCourses" 
              :key="course.id"
              class="course-distribution__movement-course"
              @click="moveStudentToCourse(movementPopoverData.student.id, course.id)"
            >
              <span class="course-distribution__movement-course-name">{{ course.name }}</span>
              <span class="course-distribution__movement-course-capacity">({{ getCourseCapacity(course) }})</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Keyboard Help Modal -->
  <div v-if="showKeyboardHelp" class="keyboard-help-modal" @click.self="toggleKeyboardHelp">
    <div class="keyboard-help-content">
      <div class="keyboard-help-header">
        <h3 class="keyboard-help-title">Keyboard Shortcuts</h3>
        <button class="keyboard-help-close" @click="toggleKeyboardHelp">×</button>
      </div>
      
      <div class="keyboard-help-body">
        <div class="keyboard-help-section">
          <h4>Navigation</h4>
          <div class="keyboard-help-item">
            <kbd>Esc</kbd>
            <span>Close popovers and modals</span>
          </div>
        </div>
        
        <div class="keyboard-help-section">
          <h4>Actions</h4>
          <div class="keyboard-help-item">
            <kbd>Ctrl+R</kbd>
            <span>Reset all students to first choice</span>
          </div>
          <div class="keyboard-help-item">
            <kbd>Ctrl+F</kbd>
            <span>Focus search input</span>
          </div>
          <div class="keyboard-help-item">
            <kbd>Ctrl+Z</kbd>
            <span>Undo last action</span>
          </div>
          <div class="keyboard-help-item">
            <kbd>Ctrl+Y</kbd>
            <span>Redo last undone action</span>
          </div>
        </div>
        
        <div class="keyboard-help-section">
          <h4>Drag & Drop</h4>
          <div class="keyboard-help-item">
            <kbd>Drag</kbd>
            <span>Drag student rows to move between courses</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { computed, ref, onMounted, onUnmounted } from 'vue';

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
    const loading = ref(false);
    const searchQuery = ref('');
    const selectedDayNumber = ref(1);
    const allStudents = ref([]);
    const allCourses = ref([]);
    const daysOfWeek = ref([]);
    const currentSemester = ref('');
    const courseNotes = ref([]);
    const visibleParticipants = ref(new Set());
    const collapsedCourseDetails = ref(new Set());
    const selectedSemesterIndex = ref(0);
    
    // Popover state management
    const activePopover = ref(null);
    const popoverPosition = ref({ x: 0, y: 0 });
    const popoverContent = ref(null);

    // Notes modal state
    const isNotesOpen = ref(false);
    const activeCourseForNotes = ref(null);
    const newNoteText = ref('');

    // Movement popover state
    const isMovementPopoverOpen = ref(false);
    const movementPopoverData = ref(null);
    const movementPopoverPosition = ref({ x: 0, y: 0 });

    // Drag and Drop state
    const draggedStudent = ref(null);
    const dragOverCourse = ref(null);
    const isDragging = ref(false);

    // Keyboard help state
    const showKeyboardHelp = ref(false);

    // Undo/Redo state
    const actionHistory = ref([]);
    const currentHistoryIndex = ref(-1);
    const maxHistorySize = 50;

    const canUndo = computed(() => currentHistoryIndex.value > 0);
    const canRedo = computed(() => currentHistoryIndex.value < actionHistory.value.length - 1);

    // Approval state
    const approvedCourseIds = ref(new Set());
    const isCourseApproved = (courseId) => approvedCourseIds.value.has(courseId);

    // Advanced filtering state
    const filterOptions = ref({
      status: 'all', // all, enrolled, waiting, going-home
      course: 'all',
      preference: 'all', // all, first, second, third
      capacity: 'all', // all, available, full, overfilled
      approved: 'all' // all, approved, not-approved
    });

    const showAdvancedFilters = ref(false);

    // Mock data for WeWeb compatibility
    const initializeMockData = () => {
      daysOfWeek.value = [
        { day_id: 1, day_number: 1, name_en: 'Monday' },
        { day_id: 2, day_number: 2, name_en: 'Tuesday' },
        { day_id: 3, day_number: 3, name_en: 'Wednesday' },
        { day_id: 4, day_number: 4, name_en: 'Thursday' },
        { day_id: 5, day_number: 5, name_en: 'Friday' }
      ];

      allCourses.value = [
        {
          id: 'course-1',
          name: 'Computer',
          teacher: 'Steffi J',
          room: 'Computer',
          day: 'Monday',
          max_capacity: 12,
          is_locked: false,
          notes_count: 0
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

      currentSemester.value = 'Fall 2024';
      loading.value = false;
    };

    // Computed properties
    const selectedDayName = computed(() => {
      const day = daysOfWeek.value.find(d => d.day_number === selectedDayNumber.value);
      return day?.name_en || 'Monday';
    });

    const searchFilteredStudents = computed(() => {
      if (!searchQuery.value.trim()) return allStudents.value;
      const query = searchQuery.value.toLowerCase();
      return allStudents.value.filter(student => 
        student.name.toLowerCase().includes(query)
      );
    });

    const filteredCourses = computed(() => {
      let courses = allCourses.value;

      // Filter by capacity
      if (filterOptions.value.capacity !== 'all') {
        courses = courses.filter(course => {
          const enrolledCount = enrolledByCourseId.value[course.id]?.length || 0;
          const status = getCourseStatus(course);
          
          switch (filterOptions.value.capacity) {
            case 'available':
              return status === 'available';
            case 'full':
              return status === 'near_full';
            case 'overfilled':
              return status === 'overfilled';
            default:
              return true;
          }
        });
      }

      // Filter by approval status
      if (filterOptions.value.approved !== 'all') {
        courses = courses.filter(course => {
          const isApproved = isCourseApproved(course.id);
          return filterOptions.value.approved === 'approved' ? isApproved : !isApproved;
        });
      }

      return courses;
    });

    const waitingListStudents = computed(() => {
      return filteredStudents.value.filter(student => student.current_enrollment === 'waiting');
    });

    const goingHomeStudents = computed(() => {
      return filteredStudents.value.filter(student => student.current_enrollment === 'going-home');
    });

    const enrolledByCourseId = computed(() => {
      const enrolled = {};
      allCourses.value.forEach(course => {
        enrolled[course.id] = filteredStudents.value.filter(student => 
          student.current_enrollment === course.id
        );
      });
      return enrolled;
    });

    const totalEnrolledForDay = computed(() => {
      return filteredStudents.value.filter(student => student.current_enrollment && student.current_enrollment !== 'waiting' && student.current_enrollment !== 'going-home').length;
    });

    const unresolvedProblems = computed(() => {
      return courseNotes.value.filter(note => note.is_problem && !note.is_resolved);
    });

    const getNotesForActiveCourse = computed(() => {
      if (!activeCourseForNotes.value) return [];
      return courseNotes.value.filter(n => n.course_id === activeCourseForNotes.value);
    });

    const notesCounts = computed(() => {
      const notes = getNotesForActiveCourse.value;
      return {
        total: notes.length,
        unresolved: notes.filter(n => !n.is_resolved).length,
        resolved: notes.filter(n => n.is_resolved).length,
      };
    });

    // Advanced filtering computed properties
    const filteredStudents = computed(() => {
      let students = allStudents.value;

      // Filter by status
      if (filterOptions.value.status !== 'all') {
        switch (filterOptions.value.status) {
          case 'enrolled':
            students = students.filter(s => s.current_enrollment && s.current_enrollment !== 'waiting');
            break;
          case 'waiting':
            students = students.filter(s => s.current_enrollment === 'waiting');
            break;
          case 'going-home':
            students = students.filter(s => s.current_enrollment === 'going-home');
            break;
        }
      }

      // Filter by course
      if (filterOptions.value.course !== 'all') {
        students = students.filter(s => s.current_enrollment === filterOptions.value.course);
      }

      // Filter by preference
      if (filterOptions.value.preference !== 'all') {
        students = students.filter(s => {
          switch (filterOptions.value.preference) {
            case 'first':
              return s.first_choice && s.current_enrollment === s.first_choice;
            case 'second':
              return s.second_choice && s.current_enrollment === s.second_choice;
            case 'third':
              return s.third_choice && s.current_enrollment === s.third_choice;
            default:
              return true;
          }
        });
      }

      return students;
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
      const course = allCourses.value.find(c => c.id === courseId);
      return course?.name || 'Unknown';
    };

    const isCurrentChoice = (student, choiceNumber) => {
      const choice = choiceNumber === 1 ? student.first_choice :
                   choiceNumber === 2 ? student.second_choice : 
                   student.third_choice;
      return student.current_enrollment === choice;
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
      const capacity = course.max_capacity;
      
      if (course.is_locked) return 'locked';
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
      return visibleParticipants.value.has(courseId);
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

    const onSemesterChange = (index) => {
      if (isEditing.value) return;
      selectedSemesterIndex.value = parseInt(index, 10);
      emit('trigger-event', { name: 'semesterChange', event: { value: selectedSemesterIndex.value } });
    };

    const onResetToFirstChoices = () => {
      if (isEditing.value) return;
      console.log('Resetting all students to their first choice...');
      allStudents.value.forEach(student => {
        student.current_enrollment = student.first_choice;
      });
      emit('trigger-event', { name: 'resetChoices' });
    };

    const onStudentMove = async (studentId, targetId) => {
      if (isEditing.value) return;
      console.log('Moving student', studentId, 'to', targetId);
      
      // Find student and get previous enrollment
      const student = allStudents.value.find(s => s.id === studentId);
      if (!student) return;
      
      const previousEnrollment = student.current_enrollment;
      
      // Update student enrollment
      student.current_enrollment = targetId;
      
      // Add to history for undo/redo
      addToHistory({
        type: 'studentMove',
        data: {
          studentId,
          previousEnrollment,
          newEnrollment: targetId,
          studentName: student.name
        }
      });
      
      emit('trigger-event', { name: 'studentMove', event: { studentId, courseId: targetId } });
    };

    const onLockToggle = async (courseId, lockState) => {
      if (isEditing.value) return;
      console.log('Toggling lock for course', courseId, 'to', lockState);
      
      // Find course and get previous lock state
      const course = allCourses.value.find(c => c.id === courseId);
      if (!course) return;
      
      const previousLockState = course.is_locked;
      
      // Update course lock state
      course.is_locked = lockState;
      
      // Add to history for undo/redo
      addToHistory({
        type: 'courseLock',
        data: {
          courseId,
          previousLockState,
          newLockState: lockState,
          courseName: course.name
        }
      });
      
      emit('trigger-event', { name: 'courseLockToggle', event: { courseId, lockState } });
    };

    const onApproveClick = (courseId) => {
      if (isEditing.value) return;
      console.log('Approving course', courseId);
      
      // Add to history for undo/redo
      addToHistory({
        type: 'courseApprove',
        data: {
          courseId,
          courseName: allCourses.value.find(c => c.id === courseId)?.name || 'Unknown Course'
        }
      });
      
      approvedCourseIds.value.add(courseId);
      emit('trigger-event', { name: 'courseApprove', event: { courseId } });
    };

    const onNotesClick = (courseId) => {
      if (isEditing.value) return;
      activeCourseForNotes.value = courseId;
      isNotesOpen.value = true;
    };

    const closeNotes = () => {
      isNotesOpen.value = false;
      activeCourseForNotes.value = null;
      newNoteText.value = '';
    };

    const addNoteForCourse = () => {
      if (!activeCourseForNotes.value || !newNoteText.value.trim()) return;
      const note = {
        id: `note-${Date.now()}`,
        course_id: activeCourseForNotes.value,
        author: 'Admin',
        text: newNoteText.value.trim(),
        created_at: new Date().toISOString(),
        is_resolved: false,
      };
      courseNotes.value.push(note);
      newNoteText.value = '';
    };

    const toggleNoteResolved = (noteId) => {
      const note = courseNotes.value.find(n => n.id === noteId);
      if (note) {
        note.is_resolved = !note.is_resolved;
        note.resolved_at = note.is_resolved ? new Date().toISOString() : null;
      }
    };

    const onResolveNote = async (noteId) => {
      if (isEditing.value) return;
      console.log('Resolving note', noteId);
      
      // Find note and get previous state
      const note = courseNotes.value.find(n => n.id === noteId);
      if (!note) return;
      
      const previousResolvedState = note.is_resolved;
      const previousResolvedAt = note.resolved_at;
      
      // Update note
      note.is_resolved = true;
      note.resolved_at = new Date().toISOString();
      
      // Add to history for undo/redo
      addToHistory({
        type: 'noteResolve',
        data: {
          noteId,
          previousResolvedState,
          previousResolvedAt,
          newResolvedState: true,
          newResolvedAt: note.resolved_at,
          noteText: note.text
        }
      });
      
      emit('trigger-event', { name: 'problemResolved', event: { noteId } });
    };

    // Popover methods
    const showPopover = (type, data, event) => {
      if (isEditing.value) return;
      
      // Calculate position
      const rect = event.target.getBoundingClientRect();
      popoverPosition.value = {
        x: rect.left + rect.width / 2,
        y: rect.bottom + 8
      };
      
      activePopover.value = type;
      popoverContent.value = data;
    };

    const hidePopover = () => {
      activePopover.value = null;
      popoverContent.value = null;
    };

    const toggleStudentPopover = (studentId, event) => {
      if (isEditing.value) return;
      
      const student = allStudents.value.find(s => s.id === studentId);
      if (!student) return;
      
      if (activePopover.value === 'student' && popoverContent.value?.id === studentId) {
        hidePopover();
      } else {
        showPopover('student', student, event);
      }
    };

    const toggleSchedulePopover = (studentId, schedule, event) => {
      if (isEditing.value) return;
      
      const student = allStudents.value.find(s => s.id === studentId);
      if (!student) return;
      
      const scheduleData = {
        student,
        schedule,
        fullSchedule: getStudentSchedule(student)
      };
      
      if (activePopover.value === 'schedule' && popoverContent.value?.student?.id === studentId) {
        hidePopover();
      } else {
        showPopover('schedule', scheduleData, event);
      }
    };

    const getStudentSchedule = (student) => {
      // Mock schedule data - in real app this would come from backend
      return [
        { day: 'Monday', period: 1, activity: 'Computer', courseId: 'course-1' },
        { day: 'Tuesday', period: 2, activity: 'Football', courseId: 'course-2' },
        { day: 'Wednesday', period: 3, activity: 'Volleyball', courseId: 'course-3' }
      ];
    };

    // Course section helper methods
    const getCourseCapacity = (course) => {
      const enrolled = enrolledByCourseId.value[course.id]?.length || 0;
      return `${enrolled}/${course.max_capacity}`;
    };

    const getCourseSectionClass = (course) => {
      const status = getCourseStatus(course);
      return {
        'course-distribution__section--overfilled': status === 'overfilled',
        'course-distribution__section--near-full': status === 'near_full',
        'course-distribution__section--locked': course.is_locked,
        'course-distribution__section--approved': isCourseApproved(course.id)
      };
    };

    const getCapacityBadgeClass = (course) => {
      const status = getCourseStatus(course);
      return {
        'course-distribution__capacity-badge--overfilled': status === 'overfilled',
        'course-distribution__capacity-badge--near-full': status === 'near_full',
        'course-distribution__capacity-badge--available': status === 'available'
      };
    };

    const getStatusBadgeClass = (course) => {
      const status = getCourseStatus(course);
      return {
        'course-distribution__status-badge--overfilled': status === 'overfilled',
        'course-distribution__status-badge--near-full': status === 'near_full',
        'course-distribution__status-badge--locked': course.is_locked,
        'course-distribution__status-badge--approved': isCourseApproved(course.id)
      };
    };

    // Notes counts per course
    const getCourseNotesCount = (courseId) => {
      const notes = courseNotes.value.filter(n => n.course_id === courseId);
      return {
        total: notes.length,
        unresolved: notes.filter(n => !n.is_resolved).length,
      };
    };

    const clearAllFilters = () => {
      filterOptions.value = {
        status: 'all',
        course: 'all',
        preference: 'all',
        capacity: 'all',
        approved: 'all'
      };
    };

    const getFilterSummary = () => {
      const activeFilters = Object.entries(filterOptions.value)
        .filter(([key, value]) => value !== 'all')
        .map(([key, value]) => `${key}: ${value}`)
        .join(', ');
      
      return activeFilters || 'No filters applied';
    };

    // Initialize data when component mounts
    onMounted(() => {
      console.log('Component mounted, initializing data...');
      initializeMockData();
      
      // Add global click listener to close popover when clicking outside
      document.addEventListener('click', (event) => {
        const popover = document.querySelector('.course-distribution__popover');
        const movementPopover = document.querySelector('.course-distribution__movement-popover');
        if (popover && !popover.contains(event.target) && activePopover.value) {
          hidePopover();
        }
        if (movementPopover && !movementPopover.contains(event.target) && isMovementPopoverOpen.value) {
          closeMovementPopover();
        }
      });
    });

    // Clean up event listener when component unmounts
    onUnmounted(() => {
      document.removeEventListener('click', hidePopover);
      document.removeEventListener('keydown', handleKeyboardShortcuts);
    });

    const openMovementPopover = (studentId, preferenceType, event) => {
      const student = allStudents.value.find(s => s.id === studentId);
      if (!student) return;

      const rect = event.target.getBoundingClientRect();
      movementPopoverPosition.value = {
        x: rect.left + rect.width / 2,
        y: rect.bottom + 8
      };

      movementPopoverData.value = {
        student,
        preferenceType,
        availableCourses: allCourses.value.filter(course => 
          !course.is_locked && 
          (enrolledByCourseId.value[course.id]?.length || 0) < course.max_capacity
        )
      };
      isMovementPopoverOpen.value = true;
    };

    const closeMovementPopover = () => {
      isMovementPopoverOpen.value = false;
      movementPopoverData.value = null;
    };

    const moveStudentToCourse = (studentId, courseId) => {
      onStudentMove(studentId, courseId);
      closeMovementPopover();
    };

    // Drag and Drop handlers
    const onDragStart = (event, student) => {
      if (isEditing.value) return;
      draggedStudent.value = student;
      isDragging.value = true;
      event.dataTransfer.effectAllowed = 'move';
      event.dataTransfer.setData('text/plain', student.id);
      
      // Add visual feedback
      event.target.style.opacity = '0.5';
    };

    const onDragEnd = (event) => {
      isDragging.value = false;
      draggedStudent.value = null;
      dragOverCourse.value = null;
      event.target.style.opacity = '1';
    };

    const onDragOver = (event, course) => {
      if (!isDragging.value || !draggedStudent.value) return;
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';
      dragOverCourse.value = course.id;
    };

    const onDragLeave = (event) => {
      dragOverCourse.value = null;
    };

    const onDrop = (event, course) => {
      if (!isDragging.value || !draggedStudent.value) return;
      event.preventDefault();
      
      // Check if course is locked or full
      if (course.is_locked) {
        console.log('Cannot drop: course is locked');
        return;
      }
      
      const enrolledCount = enrolledByCourseId.value[course.id]?.length || 0;
      if (enrolledCount >= course.max_capacity) {
        console.log('Cannot drop: course is full');
        return;
      }
      
      // Move student to course
      onStudentMove(draggedStudent.value.id, course.id);
      
      // Reset drag state
      isDragging.value = false;
      draggedStudent.value = null;
      dragOverCourse.value = null;
    };

    // Keyboard shortcuts
    const handleKeyboardShortcuts = (event) => {
      // Only handle shortcuts when not in input fields
      if (event.target.tagName === 'INPUT' || event.target.tagName === 'TEXTAREA') return;

      switch (event.key) {
        case 'Escape':
          // Close all popovers and modals
          if (activePopover.value) hidePopover();
          if (isMovementPopoverOpen.value) closeMovementPopover();
          if (isNotesOpen.value) closeNotes();
          break;
        
        case 'r':
        case 'R':
          if (event.ctrlKey || event.metaKey) {
            event.preventDefault();
            onResetToFirstChoices();
          }
          break;
        
        case 'f':
        case 'F':
          if (event.ctrlKey || event.metaKey) {
            event.preventDefault();
            document.querySelector('.course-distribution__search-input')?.focus();
          }
          break;
        
        case 'h':
        case 'H':
          if (event.ctrlKey || event.metaKey) {
            event.preventDefault();
            // Toggle help/instructions
            console.log('Help/Instructions');
          }
          break;
        
        case 'z':
        case 'Z':
          if (event.ctrlKey || event.metaKey) {
            event.preventDefault();
            undo();
          }
          break;
        
        case 'y':
        case 'Y':
          if (event.ctrlKey || event.metaKey) {
            event.preventDefault();
            redo();
          }
          break;
      }
    };

    // Add keyboard event listener
    onMounted(() => {
      document.addEventListener('keydown', handleKeyboardShortcuts);
    });

    const toggleKeyboardHelp = () => {
      showKeyboardHelp.value = !showKeyboardHelp.value;
    };

    // Undo/Redo functionality
    const addToHistory = (action) => {
      // Remove any actions after current index (for new actions after undo)
      actionHistory.value = actionHistory.value.slice(0, currentHistoryIndex.value + 1);
      
      // Add new action
      actionHistory.value.push({
        ...action,
        timestamp: Date.now(),
        id: `action-${Date.now()}-${Math.random()}`
      });
      
      // Limit history size
      if (actionHistory.value.length > maxHistorySize) {
        actionHistory.value.shift();
      } else {
        currentHistoryIndex.value++;
      }
    };

    const undo = () => {
      if (!canUndo.value) return;
      
      const action = actionHistory.value[currentHistoryIndex.value - 1];
      if (!action) return;
      
      // Revert the action
      revertAction(action);
      currentHistoryIndex.value--;
    };

    const redo = () => {
      if (!canRedo.value) return;
      
      const action = actionHistory.value[currentHistoryIndex.value + 1];
      if (!action) return;
      
      // Reapply the action
      applyAction(action);
      currentHistoryIndex.value++;
    };

    const revertAction = (action) => {
      switch (action.type) {
        case 'studentMove':
          // Revert student movement
          const student = allStudents.value.find(s => s.id === action.data.studentId);
          if (student) {
            student.current_enrollment = action.data.previousEnrollment;
          }
          break;
        case 'courseLock':
          // Revert course lock state
          const course = allCourses.value.find(c => c.id === action.data.courseId);
          if (course) {
            course.is_locked = action.data.previousLockState;
          }
          break;
        case 'courseApprove':
          // Revert course approval
          approvedCourseIds.value.delete(action.data.courseId);
          break;
        case 'noteResolve':
          // Revert note resolution
          const note = courseNotes.value.find(n => n.id === action.data.noteId);
          if (note) {
            note.is_resolved = action.data.previousResolvedState;
            note.resolved_at = action.data.previousResolvedAt;
          }
          break;
      }
    };

    const applyAction = (action) => {
      switch (action.type) {
        case 'studentMove':
          // Apply student movement
          const student = allStudents.value.find(s => s.id === action.data.studentId);
          if (student) {
            student.current_enrollment = action.data.newEnrollment;
          }
          break;
        case 'courseLock':
          // Apply course lock state
          const course = allCourses.value.find(c => c.id === action.data.courseId);
          if (course) {
            course.is_locked = action.data.newLockState;
          }
          break;
        case 'courseApprove':
          // Apply course approval
          approvedCourseIds.value.add(action.data.courseId);
          break;
        case 'noteResolve':
          // Apply note resolution
          const note = courseNotes.value.find(n => n.id === action.data.noteId);
          if (note) {
            note.is_resolved = action.data.newResolvedState;
            note.resolved_at = action.data.newResolvedAt;
          }
          break;
      }
    };

    return {
      // State
      loading,
      searchQuery,
      selectedDayNumber,
      selectedSemesterIndex,
      currentSemester,
      courseNotes,
      visibleParticipants,
      collapsedCourseDetails,
      activePopover,
      popoverPosition,
      popoverContent,
      
      // Computed
      selectedDayName,
      filteredCourses,
      waitingListStudents,
      goingHomeStudents,
      enrolledByCourseId,
      totalEnrolledForDay,
      unresolvedProblems,
      daysOfWeek,
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
      isParticipantsVisible,
      toggleParticipants,
      toggleCourseDetails,
      
      // Event handlers
      onDayChange,
      onSearchChange,
      onSemesterChange,
      onResetToFirstChoices,
      onStudentMove,
      onLockToggle,
      onApproveClick,
      onNotesClick,
      onResolveNote,
      
      // Popover methods
      showPopover,
      hidePopover,
      toggleStudentPopover,
      toggleSchedulePopover,
      getStudentSchedule,
      
      // Props
      content: props.content,

      // Notes modal
      isNotesOpen,
      activeCourseForNotes,
      newNoteText,
      getNotesForActiveCourse,
      notesCounts,
      closeNotes,
      addNoteForCourse,
      toggleNoteResolved,

      // Movement popover
      isMovementPopoverOpen,
      movementPopoverData,
      movementPopoverPosition,
      openMovementPopover,
      closeMovementPopover,
      moveStudentToCourse,

      // Drag and Drop handlers
      onDragStart,
      onDragEnd,
      onDragOver,
      onDragLeave,
      onDrop,

      // Drag state
      isDragging,
      draggedStudent,
      dragOverCourse,

      // Keyboard help state
      showKeyboardHelp,
      toggleKeyboardHelp,

      // Undo/Redo state
      actionHistory,
      currentHistoryIndex,
      maxHistorySize,
      canUndo,
      canRedo,

      // Undo/Redo functions
      undo,
      redo,
      addToHistory,

      // Advanced filtering state
      filterOptions,
      showAdvancedFilters,

      // Advanced filtering computed properties
      filteredStudents,

      // Helper functions
      clearAllFilters,
      getFilterSummary,
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
    align-items: center;
    justify-content: space-between;
    padding: 24px 32px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: 12px;
    margin-bottom: 24px;
    box-shadow: 0 4px 20px rgba(102, 126, 234, 0.15);

    &-left {
      display: flex;
      flex-direction: column;
      gap: 8px;
    }

    &-right {
      display: flex;
      align-items: center;
      gap: 16px;
    }
  }

  &__title {
    font-size: 28px;
    font-weight: 700;
    margin: 0;
    color: white;
    letter-spacing: -0.025em;
  }

  &__subtitle-container {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 16px;
    color: rgba(255, 255, 255, 0.9);
  }

  &__semester {
    font-weight: 600;
  }

  &__separator {
    color: rgba(255, 255, 255, 0.6);
  }

  &__day {
    font-weight: 500;
  }

  &__reset-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 16px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    color: white;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s ease;
    backdrop-filter: blur(10px);

    &:hover {
      background: rgba(255, 255, 255, 0.2);
      border-color: rgba(255, 255, 255, 0.3);
      transform: translateY(-1px);
    }

    &:active {
      transform: translateY(0);
    }
  }

  &__reset-icon {
    width: 16px;
    height: 16px;
  }

  &__help-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 16px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    color: white;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s ease;
    backdrop-filter: blur(10px);

    &:hover:not(:disabled) {
      background: rgba(255, 255, 255, 0.2);
      border-color: rgba(255, 255, 255, 0.3);
      transform: translateY(-1px);
    }

    &:active:not(:disabled) {
      transform: translateY(0);
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
      transform: none;
    }
  }

  &__help-icon {
    width: 16px;
    height: 16px;
  }

  &__filters-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 16px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    color: white;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s ease;
    backdrop-filter: blur(10px);

    &:hover {
      background: rgba(255, 255, 255, 0.2);
      border-color: rgba(255, 255, 255, 0.3);
      transform: translateY(-1px);
    }

    &:active {
      transform: translateY(0);
    }

    &.active {
      background: rgba(59, 130, 246, 0.3);
      border-color: rgba(59, 130, 246, 0.5);
    }
  }

  &__filters-icon {
    width: 16px;
    height: 16px;
  }

  &__filters-panel {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    margin: 20px 0;
    padding: 24px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  }

  &__filters-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
  }

  &__filters-title {
    font-size: 18px;
    font-weight: 600;
    color: #1f2937;
    margin: 0;
  }

  &__clear-filters-btn {
    padding: 8px 16px;
    background: #f3f4f6;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    color: #6b7280;
    font-size: 14px;
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover {
      background: #e5e7eb;
      color: #374151;
    }
  }

  &__filters-content {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 16px;
    margin-bottom: 20px;
  }

  &__filter-group {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__filter-label {
    font-size: 14px;
    font-weight: 500;
    color: #374151;
  }

  &__filter-select {
    padding: 10px 12px;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    background: white;
    color: #1f2937;
    font-size: 14px;
    cursor: pointer;
    transition: all 0.2s ease;

    &:focus {
      outline: none;
      border-color: #3b82f6;
      box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
  }

  &__filters-summary {
    padding: 12px 16px;
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 6px;
  }

  &__filters-text {
    font-size: 14px;
    color: #64748b;
  }

  &__search {
    position: relative;
    display: flex;
    align-items: center;
  }

  &__search-icon {
    position: absolute;
    left: 12px;
    color: #6b7280;
    pointer-events: none;
  }

  &__search-input {
    width: 280px;
    padding: 12px 12px 12px 40px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.1);
    color: white;
    font-size: 14px;
    backdrop-filter: blur(10px);
    transition: all 0.2s ease;

    &::placeholder {
      color: rgba(255, 255, 255, 0.6);
    }

    &:focus {
      outline: none;
      border-color: rgba(255, 255, 255, 0.4);
      background: rgba(255, 255, 255, 0.15);
      box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.1);
    }
  }

  &__select-wrapper {
    position: relative;
  }

  &__semester-select,
  &__day-select {
    padding: 12px 16px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.1);
    color: white;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    backdrop-filter: blur(10px);
    transition: all 0.2s ease;
    min-width: 140px;

    &:focus {
      outline: none;
      border-color: rgba(255, 255, 255, 0.4);
      background: rgba(255, 255, 255, 0.15);
      box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.1);
    }

    option {
      background: #1f2937;
      color: white;
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
  }

  // Enhanced Problems Section
  &__problems {
    background: linear-gradient(135deg, #fef2f2 0%, #fef7f0 100%);
    border: 1px solid #fecaca;
    border-radius: 16px;
    padding: 24px;
    margin-bottom: 24px;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  }

  &__problems-header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    margin-bottom: 20px;
    padding-bottom: 16px;
    border-bottom: 2px solid #fecaca;
  }

  &__problems-title {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  &__problems-icon {
    color: #dc2626;
    flex-shrink: 0;
  }

  &__problems-heading {
    font-size: 20px;
    font-weight: 700;
    color: #991b1b;
    margin: 0;
    letter-spacing: -0.025em;
  }

  &__problems-count {
    background: linear-gradient(135deg, #dc2626 0%, #b91c1c 100%);
    color: white;
    padding: 6px 12px;
    border-radius: 20px;
    font-size: 14px;
    font-weight: 700;
    box-shadow: 0 2px 4px rgba(220, 38, 38, 0.2);
  }

  &__problems-description {
    font-size: 14px;
    color: #7f1d1d;
    margin: 0;
    font-weight: 500;
  }

  &__problems-list {
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  &__problem-item {
    background: white;
    border: 1px solid #fecaca;
    border-radius: 12px;
    padding: 20px;
    transition: all 0.3s ease;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);

    &:hover {
      border-color: #fca5a5;
      box-shadow: 0 4px 12px rgba(220, 38, 38, 0.1);
      transform: translateY(-2px);
    }
  }

  &__problem-content {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__problem-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 16px;
  }

  &__problem-meta {
    display: flex;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
  }

  &__problem-course {
    font-size: 16px;
    font-weight: 700;
    color: #991b1b;
    background: #fef2f2;
    padding: 4px 12px;
    border-radius: 8px;
    border: 1px solid #fecaca;
  }

  &__problem-author {
    font-size: 14px;
    font-weight: 600;
    color: #7f1d1d;
    display: flex;
    align-items: center;
    gap: 4px;

    &::before {
      content: "by";
      font-weight: 400;
      color: #9ca3af;
    }
  }

  &__problem-date {
    font-size: 12px;
    color: #9ca3af;
    font-weight: 500;
  }

  &__problem-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__resolve-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 16px;
    background: linear-gradient(135deg, #10b981 0%, #059669 100%);
    color: white;
    border: none;
    border-radius: 10px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    box-shadow: 0 2px 4px rgba(16, 185, 129, 0.2);

    &:hover:not(:disabled) {
      background: linear-gradient(135deg, #059669 0%, #047857 100%);
      transform: translateY(-1px);
      box-shadow: 0 4px 8px rgba(16, 185, 129, 0.3);
    }

    &:active {
      transform: translateY(0);
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
      transform: none;
    }
  }

  &__resolve-icon {
    width: 16px;
    height: 16px;
    color: white;
  }

  &__problem-text {
    font-size: 15px;
    color: #374151;
    line-height: 1.6;
    margin: 0;
    padding: 12px 16px;
    background: #f9fafb;
    border-radius: 8px;
    border-left: 4px solid #fecaca;
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

  // Popover Styles
  &__popover {
    position: fixed;
    z-index: 1000;
    transform: translateX(-50%);
    pointer-events: auto;
    animation: popoverFadeIn 0.2s ease-out;
  }

  &__popover-content {
    background-color: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
    min-width: 280px;
    max-width: 320px;
    overflow: hidden;
  }

  &__popover-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 16px;
    background-color: #f9fafb;
    border-bottom: 1px solid #e5e7eb;
  }

  &__popover-title {
    font-size: 14px;
    font-weight: 600;
    color: #111827;
    margin: 0;
  }

  &__popover-close {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
    border: none;
    background: none;
    color: #6b7280;
    cursor: pointer;
    border-radius: 4px;
    font-size: 16px;
    line-height: 1;
    transition: all 0.2s ease;

    &:hover {
      background-color: #e5e7eb;
      color: #374151;
    }
  }

  &__popover-body {
    padding: 16px;
  }

  &__popover-section {
    margin-bottom: 16px;

    &:last-child {
      margin-bottom: 0;
    }
  }

  &__popover-section-title {
    font-size: 12px;
    font-weight: 600;
    color: #374151;
    margin: 0 0 8px 0;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  &__popover-text {
    font-size: 14px;
    color: #6b7280;
    margin: 0;
  }

  &__popover-preferences {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  &__popover-preference {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__popover-preference-label {
    font-size: 12px;
    font-weight: 500;
    color: #6b7280;
    min-width: 24px;
  }

  &__popover-preference-value {
    font-size: 14px;
    color: #111827;
    font-weight: 500;
  }

  &__popover-schedule {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__popover-schedule-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 6px 8px;
    border-radius: 4px;
    background-color: #f9fafb;
    transition: background-color 0.2s ease;

    &--highlighted {
      background-color: #dbeafe;
      border: 1px solid #93c5fd;
    }
  }

  &__popover-schedule-day {
    font-size: 12px;
    font-weight: 600;
    color: #374151;
    min-width: 60px;
  }

  &__popover-schedule-period {
    font-size: 11px;
    color: #6b7280;
    min-width: 30px;
  }

  &__popover-schedule-activity {
    font-size: 13px;
    color: #111827;
    font-weight: 500;
  }

  // Modern Course Sections
  &__section {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    padding: 24px;
    margin-bottom: 20px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
    transition: all 0.3s ease;

    &:hover {
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      transform: translateY(-2px);
    }

    // Drag and drop states
    &.drag-over {
      border-color: #3b82f6;
      background: #eff6ff;
      box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }

    &.drag-over-full {
      border-color: #ef4444;
      background: #fef2f2;
      box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1);
    }

    &.drag-over-locked {
      border-color: #6b7280;
      background: #f9fafb;
      box-shadow: 0 0 0 3px rgba(107, 114, 128, 0.1);
    }

    &--overfilled {
      border-color: #fecaca;
      background: linear-gradient(135deg, #fef2f2 0%, #fef7f0 100%);
    }

    &--near-full {
      border-color: #fed7aa;
      background: linear-gradient(135deg, #fffbeb 0%, #fef3c7 100%);
    }

    &--locked {
      border-color: #d1d5db;
      background: #f9fafb;
      opacity: 0.8;
    }

    &--approved {
      border-color: #bbf7d0;
      background: linear-gradient(135deg, #f0fdf4 0%, #ecfdf5 100%);
    }
  }

  &__section-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 20px;
    margin-bottom: 16px;
  }

  &__section-title {
    display: flex;
    align-items: center;
    gap: 16px;
    flex: 1;
  }

  &__course-name {
    font-size: 20px;
    font-weight: 700;
    color: #111827;
    background: none;
    border: none;
    cursor: pointer;
    padding: 8px 12px;
    border-radius: 8px;
    transition: all 0.2s ease;
    text-align: left;

    &:hover {
      background: #f3f4f6;
      color: #1f2937;
    }

    &:focus {
      outline: none;
      background: #eff6ff;
      color: #1d4ed8;
      box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
  }

  &__section-badges {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__capacity-badge {
    padding: 6px 12px;
    border-radius: 20px;
    font-size: 14px;
    font-weight: 600;
    color: white;
    background: #6b7280;

    &--overfilled {
      background: linear-gradient(135deg, #dc2626 0%, #b91c1c 100%);
      animation: pulse 2s infinite;
    }

    &--near-full {
      background: linear-gradient(135deg, #f59e0b 0%, #d97706 100%);
    }

    &--available {
      background: linear-gradient(135deg, #10b981 0%, #059669 100%);
    }
  }

  &__status-badge {
    padding: 4px 10px;
    border-radius: 12px;
    font-size: 12px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;

    &--overfilled {
      background: #fef2f2;
      color: #dc2626;
      border: 1px solid #fecaca;
    }

    &--near-full {
      background: #fffbeb;
      color: #f59e0b;
      border: 1px solid #fed7aa;
    }

    &--locked {
      background: #f3f4f6;
      color: #6b7280;
      border: 1px solid #d1d5db;
    }

    &--approved {
      background: #f0fdf4;
      color: #16a34a;
      border: 1px solid #bbf7d0;
    }
  }

  &__section-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  &__action-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    background: white;
    color: #6b7280;
    cursor: pointer;
    transition: all 0.2s ease;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
    position: relative;

    &:hover:not(:disabled) {
      border-color: #d1d5db;
      background: #f9fafb;
      color: #374151;
      transform: translateY(-1px);
      box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    }

    &:active {
      transform: translateY(0);
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
      transform: none;
    }

    &--lock {
      &:hover {
        border-color: #f59e0b;
        color: #f59e0b;
      }
    }

    &--approve {
      &:hover {
        border-color: #10b981;
        color: #10b981;
      }

      &:disabled {
        background: #10b981;
        color: white;
        border-color: #10b981;
        opacity: 1;
      }
    }

    &--notes {
      &:hover {
        border-color: #3b82f6;
        color: #3b82f6;
      }

      .course-distribution__notes-badge {
        position: absolute;
        top: -6px;
        right: -6px;
        background: #ef4444;
        color: white;
        border: 2px solid white;
        border-radius: 9999px;
        font-size: 11px;
        font-weight: 700;
        padding: 2px 6px;
        line-height: 1;
        min-width: 20px;
        text-align: center;
      }
    }

    &--expand {
      &:hover {
        border-color: #8b5cf6;
        color: #8b5cf6;
      }
    }
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
    font-weight: 600;
    color: #111827;
    margin-bottom: 4px;
    display: block;

    &--clickable {
      cursor: pointer;
      transition: color 0.2s ease;
      border-radius: 4px;
      padding: 2px 4px;
      margin: -2px -4px;

      &:hover {
        color: #2563eb;
        background-color: #eff6ff;
      }

      &:active {
        background-color: #dbeafe;
      }
    }
  }

  &__schedule-chip {
    display: inline-block;
    background-color: #f3f4f6;
    color: #6b7280;
    font-size: 11px;
    padding: 2px 6px;
    border-radius: 12px;
    margin-right: 4px;
    margin-bottom: 2px;

    &--clickable {
      cursor: pointer;
      transition: all 0.2s ease;

      &:hover {
        background-color: #dbeafe;
        color: #2563eb;
        transform: translateY(-1px);
        box-shadow: 0 2px 4px rgba(37, 99, 235, 0.1);
      }

      &:active {
        transform: translateY(0);
        box-shadow: 0 1px 2px rgba(37, 99, 235, 0.1);
      }
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

.cd-participants__content {
  overflow: hidden;
  transition: all 0.3s ease-in-out;
}

.cd-row {
  border-bottom: 1px solid #f1f5f9;
  padding: 12px 0;
  transition: background-color 0.15s ease;
  cursor: grab;

  &:hover {
    background-color: #f8fafc;
  }

  &:active {
    cursor: grabbing;
  }

  &:last-child {
    border-bottom: none;
  }

  // Drag feedback
  &.dragging {
    opacity: 0.5;
    transform: rotate(2deg);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
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

// Animations
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: .5;
  }
}

@keyframes popoverFadeIn {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-8px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

/* Notes Modal Styles */
.course-distribution__notes-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.course-distribution__notes-modal-content {
  background-color: #fff;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  width: 80%;
  max-width: 400px;
}

.course-distribution__notes-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.course-distribution__notes-modal-title {
  font-size: 18px;
  font-weight: 600;
}

.course-distribution__notes-modal-close {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #333;
}

.course-distribution__notes-modal-body {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.course-distribution__notes-modal-textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  resize: vertical;
}

.course-distribution__notes-modal-btn {
  background-color: #4CAF50;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;

  &:hover {
    background-color: #45a049;
  }
}

.notes-modal {
  position: fixed;
  inset: 0;
  background: rgba(17, 24, 39, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1100;
}

.notes-modal__content {
  width: 640px;
  max-width: 90vw;
  background: #fff;
  border-radius: 12px;
  border: 1px solid #e5e7eb;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.notes-modal__header {
  display: flex;
  align-items: center;
  gap: 12px;
  justify-content: space-between;
  padding: 16px 20px;
  background: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
}

.notes-modal__title {
  font-size: 18px;
  font-weight: 700;
  color: #111827;
  margin: 0;
}

.notes-modal__meta {
  display: flex;
  gap: 8px;
  margin-left: auto;
}

.notes-modal__count {
  font-size: 12px;
  color: #6b7280;
  background: #f3f4f6;
  border: 1px solid #e5e7eb;
  border-radius: 9999px;
  padding: 4px 8px;
}
.notes-modal__count--warn { color: #b45309; background: #fffbeb; border-color: #fef3c7; }
.notes-modal__count--ok { color: #166534; background: #f0fdf4; border-color: #bbf7d0; }

.notes-modal__close {
  margin-left: 8px;
  border: none;
  background: transparent;
  font-size: 20px;
  color: #6b7280;
  cursor: pointer;
}

.notes-modal__list {
  max-height: 360px;
  overflow: auto;
  padding: 16px 20px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.notes-item {
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 12px;
  background: #fff;
}
.notes-item--resolved {
  background: #f0fdf4;
  border-color: #bbf7d0;
}
.notes-item__top {
  display: flex;
  gap: 8px;
  align-items: center;
  margin-bottom: 6px;
}
.notes-item__course { font-weight: 700; color: #111827; }
.notes-item__author { color: #6b7280; font-size: 12px; }
.notes-item__date { color: #9ca3af; font-size: 12px; margin-left: auto; }
.notes-item__text { font-size: 14px; color: #374151; margin: 0 0 8px 0; }
.notes-item__actions { display: flex; justify-content: flex-end; }
.notes-item__btn { 
  border: 1px solid #e5e7eb; border-radius: 8px; background: #fff; color: #374151; 
  font-size: 13px; padding: 6px 10px; cursor: pointer; transition: .2s; 
}
.notes-item__btn:hover { background: #f9fafb; }

.notes-modal__empty { padding: 20px; color: #6b7280; font-style: italic; }

.notes-modal__add {
  display: flex;
  gap: 8px;
  padding: 12px 20px 16px;
  border-top: 1px solid #e5e7eb;
  background: #fafafa;
}

.notes-modal__input {
  flex: 1;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 10px 12px;
  font-size: 14px;
}

.notes-modal__add-btn {
  border: 1px solid #3b82f6;
  background: #3b82f6;
  color: #fff;
  border-radius: 8px;
  padding: 10px 14px;
  font-size: 14px;
  cursor: pointer;
}
.notes-modal__add-btn:hover { background: #2563eb; border-color: #2563eb; }

.course-distribution__movement-popover {
  position: fixed;
  z-index: 1000;
  transform: translateX(-50%);
  pointer-events: auto;
  animation: popoverFadeIn 0.2s ease-out;
}

.course-distribution__movement-content {
  background-color: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
  min-width: 280px;
  max-width: 320px;
  overflow: hidden;
}

.course-distribution__movement-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background-color: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
}

.course-distribution__movement-title {
  font-size: 14px;
  font-weight: 600;
  color: #111827;
  margin: 0;
}

.course-distribution__movement-close {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  border: none;
  background: none;
  color: #6b7280;
  cursor: pointer;
  border-radius: 4px;
  font-size: 16px;
  line-height: 1;
  transition: all 0.2s ease;

  &:hover {
    background-color: #e5e7eb;
    color: #374151;
  }
}

.course-distribution__movement-body {
  padding: 16px;
}

.course-distribution__movement-section {
  margin-bottom: 16px;

  &:last-child {
    margin-bottom: 0;
  }
}

.course-distribution__movement-section-title {
  font-size: 12px;
  font-weight: 600;
  color: #374151;
  margin: 0 0 8px 0;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.course-distribution__movement-courses {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.course-distribution__movement-course {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  border-radius: 4px;
  background-color: #f9fafb;
  transition: background-color 0.2s ease;

  &:hover {
    background-color: #dbeafe;
    border: 1px solid #93c5fd;
  }
}

.course-distribution__movement-course-name {
  font-size: 14px;
  font-weight: 600;
  color: #111827;
}

.course-distribution__movement-course-capacity {
  font-size: 12px;
  color: #6b7280;
}

/* Keyboard Help Modal Styles */
.keyboard-help-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.keyboard-help-content {
  background-color: #fff;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  width: 80%;
  max-width: 400px;
}

.keyboard-help-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.keyboard-help-title {
  font-size: 18px;
  font-weight: 600;
}

.keyboard-help-close {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #333;
}

.keyboard-help-body {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.keyboard-help-section {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.keyboard-help-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

kbd {
  background-color: #f3f4f6;
  border-radius: 4px;
  padding: 2px 4px;
  font-size: 12px;
}

.course-distribution__undo-btn,
.course-distribution__redo-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 16px;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  color: white;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  backdrop-filter: blur(10px);

  &:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.2);
    border-color: rgba(255, 255, 255, 0.3);
    transform: translateY(-1px);
  }

  &:active:not(:disabled) {
    transform: translateY(0);
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    transform: none;
  }
}

.course-distribution__undo-icon,
.course-distribution__redo-icon {
  width: 16px;
  height: 16px;
}

</style>
