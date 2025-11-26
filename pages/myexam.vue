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

#problems {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0; /* 필수! 부모가 flex일 때 overflow 작동 */
}

#problems .problems-scroll {
  flex: 1;
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
                    updateProblems();
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
                    updateProblems();
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
                    updateProblems();
                  }
                "
              />
            </fieldset>
          </div>
        </fieldset>

        <fieldset id="problems">
          <legend>문항</legend>
          <div class="problems-scroll">
            <Checkbox
              v-for="problem in problemR"
              :key="problem.id"
              :label="problem.name"
              :checked="problem.checked"
              @update:checked="
                (value) => {
                  problem.checked = value;
                  updateSelectedProblems(problem.checked, problem.name);
                }
              "
            />
          </div>
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
            <input type="text" placeholder="파일명  " id="fileTitle" />
            <button @click="download()">다운로드</button>
          </div>
        </div>
      </fieldset>
    </div>
  </div>

  <div id="imgContainer">
    <div v-for="(img, idx) in images" :key="idx" class="imgBox">
      <h3>{{ img.name }}</h3>
      <img :src="img.url" class="imgElements" />
    </div>
  </div>
</template>

<script setup>
import Sortable from 'sortablejs';
import { onMounted } from 'vue';
import JSZip from 'jszip';
import { ref } from 'vue';
import jsPDF from 'jspdf/dist/jspdf.umd.min.js';
const images = ref([]);
let selected_exams = ref([]);
import { nextTick } from 'vue';

const loadZip = async () => {
  console.log(selected_exams.value);

  const res = await fetch(serverURL + 'makemyexams', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ files: selected_exams.value }),
  });

  const zipBuffer = await res.arrayBuffer();
  const zip = await JSZip.loadAsync(zipBuffer);

  const pdf = new jsPDF('p', 'pt', 'a4');
  const pdfWidth = pdf.internal.pageSize.getWidth();
  const pdfHeight = pdf.internal.pageSize.getHeight();
  const constant = 5;

  pdf.addFileToVFS('shingraphic.ttf', shingraphic);
  // pdf.addFileToVFS('shinmiungjo.ttf', window.shinmiungjo);

  pdf.addFont('shingraphic.ttf', 'shingraphic', 'normal');
  // pdf.addFont('shinmiungjo.ttf', 'shinmiungjo', 'normal');

  let index = 0;
  for (const fileName of Object.keys(zip.files)) {
    const blob = await zip.files[fileName].async('blob');

    // Blob → DataURL
    const dataUrl = await new Promise((resolve) => {
      const reader = new FileReader();
      reader.onload = (e) => resolve(e.target.result);
      reader.readAsDataURL(blob);
    });

    // jsPDF에 바로 추가
    const img = new Image();
    await new Promise((resolve) => {
      img.onload = () => resolve();
      img.src = dataUrl;
    });

    if (index % 2 === 0 && index !== 0) pdf.addPage();

    const x =
      index % 2 === 0
        ? pdfWidth * (1 / 16) + constant
        : pdfWidth / 2 + constant;
    const y = pdfHeight * 0.1 + constant;
    const w = pdfWidth / 2 - pdfWidth * (1 / 16) - 2 * constant;
    const h = (w / img.naturalWidth) * img.naturalHeight;

    pdf.addImage(img, 'PNG', x, y, w, h);

    if (index % 2 === 0) {
      pdf.setLineWidth(1);
      pdf.setDrawColor(0, 0, 0);
      pdf.line(pdfWidth / 2, pdfHeight * 0.1, pdfWidth / 2, pdfHeight * 0.9);
      pdf.line(
        pdfWidth * (1 / 16),
        pdfHeight * 0.1,
        pdfWidth * (15 / 16),
        pdfHeight * 0.1
      );
      pdf.setFontSize(index === 0 ? 35 : 20);
      pdf.setFont('shingraphic', 'normal');
      pdf.text(fileTitle.value, pdfWidth / 2, 55, { align: 'center' });
    }

    index++;
  }

  pdf.save(fileTitle.value + '.pdf');
};
let years = ref([]);
let months = ref([]);
let subjects = ref([
  {
    name: '공통',
    checked: false,
  },
  {
    name: '미적분',
    checked: false,
  },
  {
    name: '기하',
    checked: false,
  },
  {
    name: '확률과통계',
    checked: false,
  },
]);
let problemR = ref([]);

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

let updateProblems = () => {};
let updateSelectedProblems = () => {};
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

function download() {
  loadZip();
}

fetch(serverURL + 'getmathproblems')
  .then((res) => res.json())
  .then((problems) => {
    for (let { year, month, problem_number } of problems) {
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

    updateProblems = () => {
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
      let problems_prototype = [];
      for (let { year, month, problem_number } of problems) {
        if (
          checkedYears.includes(year + '년') &&
          checkedMonths.includes(month) &&
          checkedSubjects.includes(problem_number.split('.')[1])
        ) {
          problems_prototype.push({
            name: `${year}년 ${month} 수학 ${problem_number.split('.')[1]} ${
              problem_number.split('.')[0]
            }번`,
            checked: false,
          });
        }
      }
      problemR.value = problems_prototype;
      console.log(problemR.value);
    };

    updateSelectedProblems = (checked, name) => {
      if (checked) {
        selected_exams.value.push(name);
        console.log(selected_exams);
      } else {
        selected_exams.value.splice(selected_exams.value.indexOf(name), 1);
      }
    };

    remove_exam = (exam_name) => {
      selected_exams.value.splice(selected_exams.value.indexOf(exam_name), 1);
    };

    updateExamsBySelectedExams = () => {
      problemR.value = problemR.value.map((exam) => ({
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
