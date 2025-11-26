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
#list {
  flex: 0.3; /* 아래 list 영역 꽉 채우도록 */
  min-height: 0;
  display: flex;
  flex-direction: column;
}
#filter {
  display: flex;
  flex-direction: column;
  flex: 1; /* 자동으로 크기 맞춤 */
  min-height: 0; /* 스크롤 활성화에 필요 */
  overflow-y: auto; /* 내용 넘치면 스크롤 */
  padding: 5px; /* optional, 보기 좋게 */
  white-space: pre-wrap;
}
#exams {
  flex: 1; /* filter와 자동 비율 맞춤 */
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  min-height: 0;
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

textarea {
  width: 100%; /* 고정 가로 크기 */
  height: 100%; /* 고정 세로 크기 */
  resize: none; /* 사용자가 크기 조절 불가 */
  overflow: auto; /* 내용이 넘치면 스크롤 */
  box-sizing: border-box;
}
</style>

<template>
  <div id="outer_layout">
    <Topbar></Topbar>
    <div id="layout">
      <div id="filter_exams">
        <fieldset id="filter">
          <legend>GPT command</legend>
          <button @click="copyToClipboard">전체 복사하기</button>
          {{ gptcommand }}
        </fieldset>
        <fieldset id="exams">
          <legend>exams</legend>
          <button
            @click="
              pasteFromClipboard();
              textareainputlistener();
            "
          >
            붙여넣기
          </button>
          <textarea
            v-model="textareatext"
            @input="textareainputlistener"
            ref="thetextarea"
          ></textarea>
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
          </div>
        </div>
      </fieldset>
    </div>
  </div>
</template>

<script setup>
let gptcommand = ref('');
let textareatext = ref('');
let copyToClipboard = () => {};
let textareainputlistener = () => {};

let selected_exams = ref([]);
let download_option1 = ref('1');
let download_option2 = ref('1');

let container = ref(null);
let thetextarea = ref(null);

const pasteFromClipboard = async () => {
  try {
    // 클립보드에서 텍스트 읽기
    const clipboardText = await navigator.clipboard.readText();
    textareatext.value = clipboardText; // textarea에 넣기
  } catch (err) {
    console.error('클립보드 읽기 실패:', err);
  }
};

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

onMounted(() => {
  textareainputlistener = () => {
    selected_exams.value = thetextarea.value.value
      .trim()
      .split('\n')
      .map((exam) => {
        const [year, grade, month, where, subject] = exam.split('.');
        return `${year}년 시행 ${grade} ${where} 주관 ${month} 모의고사 ${subject}`;
      });
  };
});

fetch(serverURL + 'gptcommand')
  .then((res) => res.json())
  .then((data) => {
    gptcommand.value = data.combined;
    copyToClipboard = () => {
      navigator.clipboard
        .writeText(data.combined)
        .then(() => {
          alert('복사 완료!');
        })
        .catch((err) => {
          console.error('복사 실패:', err);
        });
    };
  });
</script>
