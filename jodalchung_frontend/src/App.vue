<script setup>
import { computed, reactive, ref, watch } from "vue";

const endpoint = "/award/results/range";

const filters = reactive({
  startDate: "2025-01-20",
  endDate: "2025-03-20",
  agency: "경기도교육청",
  keyword: "늘봄",
  pageNo: 1,
  numOfRows: 20,
});

const apiRecords = ref([]);
const totalCount = ref(0);
const isLoading = ref(false);
const errorMessage = ref("");
const progress = ref({ current: 0, total: 0 });

const pad2 = (value) => String(value).padStart(2, "0");
const formatDateYmd = (date) =>
  `${date.getFullYear()}${pad2(date.getMonth() + 1)}${pad2(date.getDate())}`;

const buildDateTimeFromDate = (date, suffix) =>
  date ? `${formatDateYmd(date)}${suffix}` : "";

const latestAnnouncement = computed(() => {
  if (!apiRecords.value.length) {
    return "-";
  }
  const sorted = [...apiRecords.value].sort((a, b) =>
    (b.rgstDt || "").localeCompare(a.rgstDt || "")
  );
  return sorted[0]?.rgstDt?.slice(0, 10) || "-";
});

const normalizedRecords = computed(() =>
  apiRecords.value.map((record) => ({
    id: `${record.bidNtceNo || "-"}-${record.bidNtceOrd || "000"}`,
    title: record.bidNtceNm || "-",
    agency: record.dminsttNm || "-",
    company: record.bidwinnrNm || "-",
    amount: Number(record.sucsfbidAmt || 0),
    announcementDate: record.rgstDt ? record.rgstDt.slice(0, 10) : "-",
    awardDate: record.fnlSucsfDate || "-",
  }))
);

const filteredRecords = computed(() => {
  const start = filters.startDate ? new Date(filters.startDate) : null;
  const end = filters.endDate ? new Date(filters.endDate) : null;
  const agency = filters.agency.trim();
  const keyword = filters.keyword.trim();

  return normalizedRecords.value.filter((record) => {
    const announcement =
      record.announcementDate !== "-"
        ? parseLocalDate(record.announcementDate)
        : null;
    const inRange =
      (!start || (announcement && announcement >= start)) &&
      (!end || (announcement && announcement <= end));
    const agencyMatch = !agency || record.agency.includes(agency);
    const keywordMatch =
      !keyword ||
      record.title.includes(keyword) ||
      record.company.includes(keyword) ||
      record.id.includes(keyword);

    return inRange && agencyMatch && keywordMatch;
  });
});

const totalPages = computed(() => {
  const size = Math.max(1, Number(filters.numOfRows) || 1);
  return Math.max(1, Math.ceil(filteredRecords.value.length / size));
});

const pagedRecords = computed(() => {
  const size = Math.max(1, Number(filters.numOfRows) || 1);
  const page = Math.min(Math.max(1, filters.pageNo), totalPages.value);
  const startIndex = (page - 1) * size;
  return filteredRecords.value.slice(startIndex, startIndex + size);
});

const companySummaries = computed(() => {
  const summaryMap = new Map();

  filteredRecords.value.forEach((record) => {
    const entry = summaryMap.get(record.company) || {
      company: record.company,
      count: 0,
      agencies: new Map(),
    };
    entry.count += 1;
    entry.agencies.set(
      record.agency,
      (entry.agencies.get(record.agency) || 0) + 1
    );
    summaryMap.set(record.company, entry);
  });

  return Array.from(summaryMap.values())
    .map((entry) => {
      const topAgencies = Array.from(entry.agencies.entries())
        .sort((a, b) => b[1] - a[1])
        .slice(0, 3)
        .map(([name, count], index) => ({
          rank: index + 1,
          name,
          count,
        }));

      return {
        company: entry.company,
        count: entry.count,
        topAgencies,
      };
    })
    .sort((a, b) => b.count - a.count);
});

watch(
  () => [filteredRecords.value.length, filters.numOfRows],
  () => {
    const maxPage = totalPages.value;
    if (filters.pageNo > maxPage) {
      filters.pageNo = maxPage;
    }
    if (filters.pageNo < 1) {
      filters.pageNo = 1;
    }
  }
);

const canSearch = computed(
  () => filters.startDate.trim() && filters.endDate.trim()
);

const parseLocalDate = (value) => {
  if (!value) return null;
  const [year, month, day] = value.split("-").map(Number);
  if (!year || !month || !day) return null;
  return new Date(year, month - 1, day);
};

const validateDateRange = () => {
  const start = parseLocalDate(filters.startDate);
  const end = parseLocalDate(filters.endDate);
  if (!start || !end) {
    return "조회 기간을 올바르게 입력해 주세요.";
  }
  if (end < start) {
    return "종료일이 시작일보다 이전입니다.";
  }

  const todayLocal = new Date();
  todayLocal.setHours(0, 0, 0, 0);
  if (end > todayLocal) {
    return "조회 기간은 오늘 이전 날짜만 가능합니다.";
  }

  const diffDays = Math.floor((end - start) / (1000 * 60 * 60 * 24));
  if (diffDays > 365 * 5) {
    return "조회 기간은 최대 5년까지 가능합니다.";
  }

  return "";
};

const extractItems = (payload) => {
  const rawItems = payload?.items || [];
  return Array.isArray(rawItems) ? rawItems : [rawItems];
};

const buildQuery = (start, end) => {
  const params = new URLSearchParams({
    inqryBgnDt: buildDateTimeFromDate(start, "0000"),
    inqryEndDt: buildDateTimeFromDate(end, "2359"),
  });

  if (filters.agency.trim()) {
    params.set("dminsttNm", filters.agency.trim());
  }

  if (filters.keyword.trim()) {
    params.set("bidNtceNm", filters.keyword.trim());
  }

  return params.toString();
};

const fetchRecords = async () => {
  if (isLoading.value) {
    return;
  }
  if (!canSearch.value) {
    errorMessage.value = "조회 기간을 입력해 주세요.";
    return;
  }

  const validationMessage = validateDateRange();
  if (validationMessage) {
    errorMessage.value = validationMessage;
    return;
  }

  isLoading.value = true;
  errorMessage.value = "";
  progress.value = { current: 0, total: 1 };

  try {
    const start = parseLocalDate(filters.startDate);
    const end = parseLocalDate(filters.endDate);
    const query = buildQuery(start, end);
    const response = await fetch(`${endpoint}?${query}`, {
      method: "GET",
    });
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    const result = await response.json();
    if (result?.resultCode && result.resultCode !== "00") {
      throw new Error(result.resultMsg || "API error");
    }

    apiRecords.value = extractItems(result);
    totalCount.value = Number(result?.totalCount || apiRecords.value.length);
    filters.pageNo = 1;
    progress.value.current = 1;
  } catch (error) {
    errorMessage.value = `Fetch failed: ${error.message || "Network error"}`;
    apiRecords.value = [];
    totalCount.value = 0;
  } finally {
    isLoading.value = false;
  }
};

const formatAmount = (value) =>
  new Intl.NumberFormat("ko-KR").format(value);
</script>

<template>
  <div class="page">
    <header class="hero">
      <div class="hero__text">
        <p class="hero__tag">나라장터 낙찰 기록 인사이트</p>
        <h1>입찰 결과 기반 낙찰 업체 분석</h1>
        <p class="hero__desc">
          공고 게시기간, 수요기관, 키워드를 입력하면 낙찰 기록을 조회하고
          업체별 낙찰 건수와 주요 수요기관 TOP3를 한눈에 확인할 수 있습니다.
        </p>
      </div>
      <div class="hero__panel">
        <div class="metric">
          <p class="metric__label">현재 필터 결과</p>
          <p class="metric__value">{{ filteredRecords.length }}건</p>
        </div>
        <div class="metric">
          <p class="metric__label">분석 대상 업체</p>
          <p class="metric__value">{{ companySummaries.length }}곳</p>
        </div>
        <div class="metric">
          <p class="metric__label">가장 최근 공고</p>
          <p class="metric__value">{{ latestAnnouncement }}</p>
        </div>
      </div>
    </header>

    <section class="filters">
      <div class="filters__header">
        <h2>조회 조건</h2>
        <p>조회 기간과 수요기관, 키워드를 입력해 검색하세요.</p>
      </div>
      <div class="filters__grid">
        <label class="input">
          <span>시작일</span>
          <input type="date" v-model="filters.startDate" />
        </label>
        <label class="input">
          <span>종료일</span>
          <input type="date" v-model="filters.endDate" />
        </label>
        <label class="input">
          <span>수요기관</span>
          <input
            type="text"
            v-model="filters.agency"
            placeholder="예: 경기도교육청"
          />
        </label>
        <label class="input">
          <span>키워드</span>
          <input type="text" v-model="filters.keyword" placeholder="공고명, 업체명, ID" />
        </label>
      </div>
      <div class="filters__actions">
        <button class="button" type="button" :disabled="isLoading" @click="fetchRecords">
          {{ isLoading ? "조회 중..." : "조회하기" }}
        </button>
        <div class="progress" v-if="isLoading && progress.total">
          <div class="progress__bar">
            <span
              class="progress__fill"
              :style="{ width: `${Math.round((progress.current / progress.total) * 100)}%` }"
            ></span>
          </div>
          <span class="progress__text">
            {{ Math.round((progress.current / progress.total) * 100) }}% ({{
              progress.current
            }}/{{ progress.total }})
          </span>
        </div>
        <p class="status" v-if="totalCount">API 총 {{ totalCount }}건</p>
        <p class="status status--error" v-if="errorMessage">{{ errorMessage }}</p>
      </div>
    </section>

    <section class="content">
      <div class="panel">
        <div class="panel__header">
          <h2>필터링 결과</h2>
          <p>나라장터 API 반환값 중 주요 필드를 추출했습니다.</p>
        </div>
        <div class="table">
          <div class="table__head">
            <span>공고명</span>
            <span>수요기관</span>
            <span>낙찰업체</span>
            <span>낙찰금액</span>
            <span>공고일</span>
            <span>낙찰일</span>
          </div>
          <div class="table__body">
            <div v-for="record in pagedRecords" :key="record.id" class="table__row">
              <span class="title">{{ record.title }}</span>
              <span>{{ record.agency }}</span>
              <span>{{ record.company }}</span>
              <span>{{ formatAmount(record.amount) }}원</span>
              <span>{{ record.announcementDate }}</span>
              <span>{{ record.awardDate }}</span>
            </div>
            <div v-if="isLoading" class="table__empty">조회 중입니다.</div>
            <div v-else-if="filteredRecords.length === 0" class="table__empty">
              조건에 맞는 낙찰 기록이 없습니다.
            </div>
          </div>
        </div>
        <div class="pagination" v-if="filteredRecords.length">
          <button
            class="button button--ghost"
            type="button"
            :disabled="filters.pageNo <= 1"
            @click="filters.pageNo -= 1"
          >
            이전
          </button>
          <span class="pagination__info">
            {{ filters.pageNo }} / {{ totalPages }}
          </span>
          <button
            class="button button--ghost"
            type="button"
            :disabled="filters.pageNo >= totalPages"
            @click="filters.pageNo += 1"
          >
            다음
          </button>
        </div>
        <div class="panel panel--secondary">
          <div class="panel__header">
            <h2>업체별 낙찰 집계</h2>
            <p>필터링 결과를 기준으로 업체별 낙찰 건수와 주요 수요기관 TOP3</p>
          </div>
          <div class="table table--summary">
            <div class="table__head">
              <span>업체명</span>
              <span>낙찰건수</span>
              <span>TOP1 수요기관</span>
              <span>TOP2 수요기관</span>
              <span>TOP3 수요기관</span>
            </div>
            <div class="table__body">
              <div v-for="summary in companySummaries" :key="summary.company" class="table__row">
                <span class="title">{{ summary.company }}</span>
                <span>{{ summary.count }}건</span>
                <span>
                  {{
                    summary.topAgencies[0]
                      ? `${summary.topAgencies[0].name} (${summary.topAgencies[0].count}건)`
                      : "-"
                  }}
                </span>
                <span>
                  {{
                    summary.topAgencies[1]
                      ? `${summary.topAgencies[1].name} (${summary.topAgencies[1].count}건)`
                      : "-"
                  }}
                </span>
                <span>
                  {{
                    summary.topAgencies[2]
                      ? `${summary.topAgencies[2].name} (${summary.topAgencies[2].count}건)`
                      : "-"
                  }}
                </span>
              </div>
              <div v-if="isLoading" class="table__empty">집계 중입니다.</div>
              <div v-else-if="companySummaries.length === 0" class="table__empty">
                집계 가능한 업체가 없습니다.
              </div>
            </div>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
  font-family: "Pretendard", "SUIT", "Apple SD Gothic Neo", sans-serif;
  background: radial-gradient(circle at top, #f6f0df 0%, #f2f5f8 45%, #ebeff3 100%);
  color: #1d2026;
}

.page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 48px 24px 80px;
  display: flex;
  flex-direction: column;
  gap: 32px;
}

.hero {
  display: grid;
  grid-template-columns: minmax(0, 1.2fr) minmax(0, 0.8fr);
  gap: 32px;
  align-items: center;
  background: linear-gradient(120deg, #101f2f 0%, #1e2f44 40%, #cdd6e1 100%);
  color: #f9f5ef;
  padding: 36px;
  border-radius: 28px;
  box-shadow: 0 24px 45px rgba(16, 25, 38, 0.2);
  overflow: hidden;
}

.hero__text h1 {
  font-size: clamp(28px, 3.5vw, 44px);
  margin: 0 0 12px;
  letter-spacing: -0.02em;
}

.hero__tag {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.16em;
  margin-bottom: 12px;
  color: #ffd79a;
}

.hero__desc {
  margin: 0;
  font-size: 16px;
  color: #e3e5ec;
}

.hero__panel {
  display: grid;
  gap: 16px;
}

.metric {
  background: rgba(255, 255, 255, 0.12);
  border-radius: 18px;
  padding: 18px;
  backdrop-filter: blur(6px);
}

.metric__label {
  margin: 0;
  font-size: 12px;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #ffe6c7;
}

.metric__value {
  margin: 8px 0 0;
  font-size: 22px;
  font-weight: 700;
}

.filters {
  background: #fff7eb;
  border-radius: 22px;
  padding: 24px;
  box-shadow: 0 12px 30px rgba(31, 40, 55, 0.08);
}

.filters__header h2 {
  margin: 0 0 6px;
  font-size: 20px;
}

.filters__header p {
  margin: 0;
  color: #5a5f66;
}

.filters__grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
  margin-top: 20px;
}

.input {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-size: 13px;
  color: #33383f;
}

.input small {
  color: #7b7f86;
  font-size: 11px;
}

.input input {
  padding: 12px 14px;
  border-radius: 12px;
  border: 1px solid #d7cbbd;
  background: #fffdf9;
  font-size: 14px;
  outline: none;
  transition: border 0.2s ease, box-shadow 0.2s ease;
}

.input input:focus {
  border-color: #1d2026;
  box-shadow: 0 0 0 3px rgba(38, 50, 68, 0.15);
}

.filters__actions {
  margin-top: 18px;
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}

.progress {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 220px;
}

.progress__bar {
  width: 140px;
  height: 8px;
  background: #efe6d7;
  border-radius: 999px;
  overflow: hidden;
}

.progress__fill {
  display: block;
  height: 100%;
  background: #1d2026;
  transition: width 0.2s ease;
}

.progress__text {
  font-size: 12px;
  color: #5a5f66;
}

.button {
  border: none;
  border-radius: 999px;
  padding: 12px 22px;
  background: #1d2026;
  color: #fff;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.button:disabled {
  opacity: 0.6;
  cursor: default;
  transform: none;
  box-shadow: none;
}

.button:not(:disabled):hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px rgba(29, 32, 38, 0.2);
}

.button--ghost {
  background: #fff;
  color: #1d2026;
  border: 1px solid #d7cbbd;
  box-shadow: none;
}

.button--ghost:not(:disabled):hover {
  box-shadow: 0 8px 16px rgba(29, 32, 38, 0.12);
}

.status {
  margin: 0;
  font-size: 13px;
  color: #5a5f66;
}

.status--error {
  color: #b1332b;
  font-weight: 600;
}

.content {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.panel {
  background: #fff;
  border-radius: 20px;
  padding: 24px;
  box-shadow: 0 18px 40px rgba(18, 29, 44, 0.08);
}

.panel--secondary {
  margin-top: 24px;
  background: #fdfbf7;
  border: 1px solid #efe6d7;
}

.panel__header h2 {
  margin: 0 0 6px;
  font-size: 20px;
}

.panel__header p {
  margin: 0 0 18px;
  color: #5a5f66;
}

.table {
  display: grid;
  gap: 12px;
}

.table__head,
.table__row {
  display: grid;
  grid-template-columns: 2fr 1.1fr 1fr 1fr 0.9fr 0.9fr;
  gap: 12px;
  align-items: center;
  font-size: 13px;
}

.table__head {
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-size: 11px;
  color: #7b7f86;
}

.table__row {
  padding: 14px 12px;
  border-radius: 14px;
  background: #f6f7fa;
}

.table__row .title {
  font-weight: 600;
}

.table__empty {
  padding: 20px;
  text-align: center;
  color: #7b7f86;
}

.table--summary .table__row,
.table--summary .table__head {
  grid-template-columns: 1.2fr 0.6fr 1fr 1fr 1fr;
}

.pagination {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 12px;
}

.pagination__info {
  font-size: 13px;
  color: #5a5f66;
}

@media (max-width: 960px) {
  .hero {
    grid-template-columns: 1fr;
  }

  .table__head,
  .table__row {
    grid-template-columns: 1.8fr 1fr 1fr;
  }

  .table__row span:nth-child(4),
  .table__row span:nth-child(5),
  .table__row span:nth-child(6),
  .table__head span:nth-child(4),
  .table__head span:nth-child(5),
  .table__head span:nth-child(6) {
    display: none;
  }

  .table--summary .table__head,
  .table--summary .table__row {
    grid-template-columns: 1.4fr 0.8fr 1.2fr;
  }

  .table--summary .table__row span:nth-child(4),
  .table--summary .table__row span:nth-child(5),
  .table--summary .table__head span:nth-child(4),
  .table--summary .table__head span:nth-child(5) {
    display: none;
  }
}

@media (max-width: 640px) {
  .page {
    padding: 32px 16px 64px;
  }

  .hero {
    padding: 28px;
  }

  .filters,
  .panel {
    padding: 20px;
  }
}
</style>
