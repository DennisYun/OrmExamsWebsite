<style scoped>
#outer_layout {
  display: flex;
  flex-direction: column;
  height: 100vh;
  box-sizing: border-box;
}

#layout {
  display: flex;
  flex-direction: column; /* 위 filter+exams, 아래 list */
  flex: 1; /* 남은 세로 다 차지 */
  min-height: 0;
  gap: 10px;
}

#filter_exams {
  display: flex;
  gap: 10px;
  flex: 1;
  min-height: 0; /* 스크롤 활성화에 필요 */
}

#filter {
  display: flex;
  flex-direction: column;
  flex: 1; /* 자동으로 크기 맞춤 */
  min-height: 0;
}

#filter_grid {
  display: flex;
  gap: 10px;
  flex: 1;
  min-height: 0;
}

#filter_grid fieldset {
  flex: 1; /* 자동 비율 분배 */
  /* display: flex; */
  flex-direction: column;
  overflow-y: auto; /* 스크롤 가능 */
  min-height: 0;
}

#exams {
  flex: 1; /* filter와 자동 비율 맞춤 */
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  min-height: 0;
}

.exam_box {
  border: 1px solid black;
  display: flex;
  padding: 5px;
  cursor: pointer;
}

#list {
  flex: 0.3; /* 아래 list 영역 꽉 채우도록 */
  min-height: 0;
  display: flex;
  flex-direction: column;
}

#list_layout {
  display: flex;
  flex-direction: column;
  flex: 1;
}

#list_layout div:nth-child(1) {
  flex: 1;
  overflow-y: auto;
}

#download_button {
  display: flex;
  justify-content: right;
}

.selected_exam {
  border: 1px solid black;
  display: flex;
  justify-content: space-between;
  padding: 5px;
  max-width: 200px;
}

#list_inner_layout {
  display: flex;
}
</style>

<template>
  <div id="outer_layout">
    <Topbar></Topbar>
    <div id="layout">
      <!-- filter + exams 가로 -->
      <div id="filter_exams">
        <fieldset id="filter">
          <legend>filter</legend>
          <div id="filter_grid">
            <fieldset>
              <legend>시행년도</legend>
              <Checkbox
                label="모두 선택"
                :checked="select_all[0].checked"
                @update:checked="(value) => (select_all[0].checked = value)"
              ></Checkbox>
              <Checkbox
                v-for="year in years"
                :key="year.id"
                :label="year.name"
                :checked="year.checked"
                @update:checked="
                  (value) => {
                    year.checked = value;
                    updatesubject();
                    updateExams();
                  }
                "
              />
            </fieldset>
            <fieldset>
              <legend>시행월</legend>
              <Checkbox
                label="모두 선택"
                :checked="select_all[1].checked"
                @update:checked="(value) => (select_all[1].checked = value)"
              ></Checkbox>
              <Checkbox
                v-for="month in months"
                :key="month.id"
                :label="month.name"
                :checked="month.checked"
                @update:checked="
                  (value) => {
                    month.checked = value;
                    updatesubject();
                    updateExams();
                  }
                "
              />
            </fieldset>
            <fieldset>
              <legend>과목</legend>
              <Checkbox
                label="모두 선택"
                :checked="select_all[2].checked"
                @update:checked="(value) => (select_all[2].checked = value)"
              ></Checkbox>
              <Checkbox
                v-for="subject in subjects"
                :key="subject.id"
                :label="subject.name"
                :checked="subject.checked"
                @update:checked="
                  (value) => {
                    subject.checked = value;
                    updateExams();
                    updateExamsBySelectedExams();
                  }
                "
              />
            </fieldset>
          </div>
        </fieldset>

        <fieldset id="exams">
          <legend>exams</legend>
          <Checkbox
            v-for="exam in examsR"
            :key="exam.id"
            :label="exam.name"
            :checked="exam.checked"
            @update:checked="
              (value) => {
                exam.checked = value;
                updateSelectedExams(exam.checked, exam.name);
              }
            "
          />
        </fieldset>
      </div>

      <fieldset id="list">
        <legend>list</legend>
        <div id="list_layout">
          <div id="list_inner_layout" ref="container">
            <div
              class="selected_exam"
              v-for="selected_exam in selected_exams"
              :key="selected_exam.id"
              @click="
                remove_exam(selected_exam);
                updateExamsBySelectedExams();
              "
            >
              <div>{{ selected_exam }}</div>
              <span class="material-icons">delete</span>
            </div>
          </div>
          <div id="download_button">
            <select v-model="download_option1">
              <option value="1">문제</option>
              <option value="2">정답과 해설</option>
              <option value="3">정답지</option>
              <option value="4">문제 + 정답과 해설</option>
              <option value="5">문제 + 정답지</option>
              <option value="6">문제 + 정답과 해설 + 정답지</option>
            </select>
            <select v-model="download_option2">
              <option value="1">~년 시행 고3 ~주관 ~월 모의고사 ~</option>
              <option value="2">~학년도 고3 ~주관 ~월 모의고사 ~</option>
              <option value="3">~년 시행 고3 ~월 모의고사 ~</option>
              <option value="4">~학년도 고3 ~월 모의고사 ~</option>
              <option value="5">~.~.~(시행년)</option>
              <option value="6">~.~.~(학년도)</option>
            </select>
            <button @click="download()">다운로드</button>
            <button @click="merge()">병합 후 다운로드</button>
          </div>
        </div>
      </fieldset>
    </div>
  </div>
</template>

<script setup>
import Sortable from 'sortablejs';
import { onMounted } from 'vue';

let years = ref([]);
let months = ref([]);
let subjects = ref([]);
let examsR = ref([]);
let selected_exams = ref([]);

let download_option1 = ref('1');
let download_option2 = ref('1');

let container = ref(null);

onMounted(() => {
  Sortable.create(container.value, {
    animation: 150,
    ghostClass: 'dragging',
    onEnd: (evt) => {
      // 배열 순서 변경
      const movedItem = selected_exams.value.splice(evt.oldIndex, 1)[0];
      selected_exams.value.splice(evt.newIndex, 0, movedItem);
      updateExamsBySelectedExams();
    },
  });
});

let updatesubject = () => {};
let updateExams = () => {};
let updateSelectedExams = () => {};
let remove_exam = () => {};
let updateExamsBySelectedExams = () => {};

function parseExamTitle(title) {
  // "년 시행" 기준으로 나누기
  let [year, rest1] = title.split('년 시행');
  year = year.trim();

  // "주관" 기준으로 나누기
  let [gradeWhere, rest2] = rest1.split('주관');
  gradeWhere = gradeWhere.trim();

  // "모의고사" 기준으로 나누기
  let [month, subject] = rest2.split('모의고사');
  month = month.trim();
  subject = subject.trim();

  // gradeWhere를 다시 쪼개서 학년 + where 구분
  let [grade, where] = gradeWhere.split(' ');

  return {
    year,
    grade,
    where,
    month,
    subject,
  };
}

function download_file(original_name, file_name) {
  fetch(serverURL + 'download/' + encodeURIComponent(original_name))
    .then((res) => {
      if (!res.ok) {
        throw new Error('파일 다운로드 실패');
      }
      return res.blob();
    })
    .then((blob) => {
      const link = document.createElement('a');
      const objectURL = URL.createObjectURL(blob);

      link.href = objectURL;
      link.download = file_name;
      document.body.appendChild(link);
      link.click();

      document.body.removeChild(link);
      URL.revokeObjectURL(objectURL);
    })
    .catch((err) => {
      console.error(err);
    });
}

function download() {
  for (let selected_exam of selected_exams.value) {
    const { year, grade, where, month, subject } =
      parseExamTitle(selected_exam);
    const original_name = `${year}.${grade}.${month}.${where}.${subject}`;
    let file_name = '';

    switch (download_option2.value) {
      case '1':
        file_name = `${year}년 시행 고3 ${where} 주관 ${month} 모의고사 ${subject}`;
        break;
      case '2':
        file_name = `${year}학년도 고3 ${where} 주관 ${month} 모의고사 ${subject}`;
        break;
      case '3':
        file_name = `${year}년 시행 고3 ${month} 모의고사 ${subject}`;
        break;
      case '4':
        file_name = `${year}학년도 고3 ${month} 모의고사 ${subject}`;
        break;
      case '5':
        file_name = `${year}.${month}.${subject}`;
        break;
      case '6':
        file_name = `${year}.${month}.${subject}`;
        break;
    }

    switch (download_option1.value) {
      case '1':
        download_file(original_name + '.pdf', file_name + '.pdf');
        break;
      case '2':
        download_file(
          original_name + '@해설.pdf',
          file_name + ' 정답과 해설.pdf'
        );
        break;
      case '3':
        download_file(
          original_name + '@해설.pdf',
          file_name + ' 정답과 해설.pdf'
        );
        break;
      case '4':
        download_file(original_name + '.pdf', file_name + '.pdf');
        download_file(
          original_name + '@해설.pdf',
          file_name + ' 정답과 해설.pdf'
        );
        break;
      case '5':
        download_file(original_name + '.pdf', file_name + '.pdf');
        download_file(
          original_name + '@해설.pdf',
          file_name + ' 정답과 해설.pdf'
        );
        break;
      case '6':
        download_file(original_name + '.pdf', file_name + '.pdf');
        download_file(
          original_name + '@해설.pdf',
          file_name + ' 정답과 해설.pdf'
        );
        break;
    }
  }
}

function merge() {
  let merge_exam_list = [];
  for (let selected_exam of selected_exams.value) {
    const { year, grade, where, month, subject } =
      parseExamTitle(selected_exam);
    const original_name = `${year}.${grade}.${month}.${where}.${subject}`;

    switch (download_option1.value) {
      case '1':
        merge_exam_list.push(original_name + '.pdf');
        break;
      case '2':
        merge_exam_list.push(original_name + '@해설.pdf');
        break;
      case '3':
        merge_exam_list.push(original_name + '@해설.pdf');
        break;
      case '4':
        merge_exam_list.push(original_name + '.pdf');
        merge_exam_list.push(original_name + '@해설.pdf');
        break;
      case '5':
        merge_exam_list.push(original_name + '.pdf');
        merge_exam_list.push(original_name + '@해설.pdf');
        break;
      case '6':
        merge_exam_list.push(original_name + '.pdf');
        merge_exam_list.push(original_name + '@해설.pdf');
        break;
    }
  }

  fetch(serverURL + 'merge', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      files: merge_exam_list,
    }),
  })
    .then((res) => {
      if (!res.ok) {
        throw new Error('파일 다운로드 실패');
      }
      return res.blob();
    })
    .then((blob) => {
      const link = document.createElement('a');
      const objectURL = URL.createObjectURL(blob);

      link.href = objectURL;
      link.download = 'merged_exams.pdf';
      document.body.appendChild(link);
      link.click();

      document.body.removeChild(link);
      URL.revokeObjectURL(objectURL);
    })
    .catch((err) => {
      console.error(err);
    });
}

fetch(serverURL + 'getexams')
  .then((res) => res.json())
  .then((exams) => {
    for (let { year, month, subject, where } of exams) {
      if (!years.value.includes(year + '년')) {
        years.value.push(year + '년');
      }
      if (!months.value.includes(month)) {
        months.value.push(month);
      }
    }
    years.value = years.value.map((value) => ({ name: value, checked: false }));
    months.value = months.value
      .sort((a, b) => {
        const numA = parseInt(a);
        const numB = parseInt(b);

        // 숫자가 없는 경우(수능 등) 처리
        if (isNaN(numA)) return 1; // 숫자 있는 게 앞
        if (isNaN(numB)) return -1;

        return numA - numB;
      })
      .map((value) => ({
        name: value,
        checked: false,
      }));

    updatesubject = () => {
      const checkedYears = years.value
        .filter((y) => y.checked)
        .map((y) => y.name);
      const checkedMonths = months.value
        .filter((m) => m.checked)
        .map((m) => m.name);

      let filteredSubjects = [];
      for (let { year, month, subject } of exams) {
        if (
          checkedYears.includes(year + '년') &&
          checkedMonths.includes(month) &&
          !filteredSubjects.includes(subject)
        ) {
          filteredSubjects.push(subject);
        }
      }
      subjects.value = filteredSubjects.map((subject) => ({
        name: subject,
        checked: false,
      }));
      console.log(subjects.value);
    };

    updateExams = () => {
      console.log('exam updated');
      const checkedYears = years.value
        .filter((y) => y.checked)
        .map((y) => y.name);
      const checkedMonths = months.value
        .filter((m) => m.checked)
        .map((m) => m.name);
      const checkedSubjects = subjects.value
        .filter((s) => s.checked)
        .map((s) => s.name);

      let filteredExams = [];
      for (let { year, month, subject, where } of exams) {
        if (
          checkedYears.includes(year + '년') &&
          checkedMonths.includes(month) &&
          checkedSubjects.includes(subject)
        ) {
          filteredExams.push(
            `${year}년 시행 고3 ${where} 주관 ${month} 모의고사 ${subject}`
          );
        }
      }
      examsR.value = filteredExams.map((exam) => ({
        name: exam,
        checked: false,
      }));
      console.log(examsR.value);
    };

    updateSelectedExams = (checked, name) => {
      if (checked) {
        selected_exams.value.push(name);
      } else {
        selected_exams.value.splice(selected_exams.value.indexOf(name), 1);
      }
    };

    remove_exam = (exam_name) => {
      selected_exams.value.splice(selected_exams.value.indexOf(exam_name), 1);
    };

    updateExamsBySelectedExams = () => {
      examsR.value = examsR.value.map((exam) => ({
        name: exam.name,
        checked: selected_exams.value.includes(exam.name),
      }));
    };
  });

let select_all = ref([
  { checked: false },
  { checked: false },
  { checked: false },
]);
</script>
