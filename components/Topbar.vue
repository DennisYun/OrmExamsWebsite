<style scoped>
#functions {
  border: 1px solid black;
  flex: 1;
  display: flex;
  align-items: center;
  gap: 20px;
}

#topbar_layout {
  display: flex;
}
</style>

<template>
  <div id="topbar_layout">
    <h1>Orm exams</h1>
    <div id="functions">
      <a href="/download">다운로드</a>
      <a href="/aidownload">다운로드 with AI</a>
      <a href="/myexam">나만의 시험지</a>
      <a href="/findenglish">영어 검색</a>
      <a href="/findkorean">국어 검색</a>
    </div>
  </div>
</template>
