<script setup>
import { computed, reactive, ref, watch } from 'vue';

const apiBase = import.meta.env.VITE_API_BASE || '';
const endpoint = apiBase
  ? `${apiBase.replace(/\/$/, '')}/award/results/range`
  : '/award/results/range';

const filters = reactive({
  startDate: '2025-01-20',
  endDate: '2025-03-20',
  agency: '경기도교육청',
  keyword: '늘봄',
  pageNo: 1,
  numOfRows: 20,
});

const apiRecords = ref([]);
const totalCount = ref(0);
const isLoading = ref(false);
const errorMessage = ref('');
const progress = ref({ current: 0, total: 0 });
const resultSearch = ref('');

const pad2 = (value) => String(value).padStart(2, '0');
const formatDateYmd = (date) =>
  `${date.getFullYear()}${pad2(date.getMonth() + 1)}${pad2(date.getDate())}`;

const buildDateTimeFromDate = (date, suffix) =>
  date ? `${formatDateYmd(date)}${suffix}` : '';

const normalizedRecords = computed(() =>
  apiRecords.value.map((record) => ({
    id: `${record.bidNtceNo || '-'}-${record.bidNtceOrd || '000'}`,
    title: record.bidNtceNm || '-',
    agency: record.dminsttNm || '-',
    company: record.bidwinnrNm || '-',
    companyAddress: record.bidwinnrAdrs || '-',
    companyCeo: record.bidwinnrCeoNm || '-',
    companyTel: record.bidwinnrTelNo || '-',
    amount: Number(record.sucsfbidAmt || 0),
    announcementDate: record.rgstDt ? record.rgstDt.slice(0, 10) : '-',
    awardDate: record.fnlSucsfDate || '-',
  })),
);

const filterColumns = [
  { key: 'title', label: '공고명', width: '2fr' },
  { key: 'agency', label: '수요기관', width: '1.1fr' },
  { key: 'company', label: '낙찰업체', width: '1fr' },
  { key: 'amount', label: '낙찰금액', width: '1fr' },
  { key: 'announcementDate', label: '공고일', width: '0.9fr' },
  { key: 'awardDate', label: '낙찰일', width: '0.9fr' },
];

const columnVisibility = reactive(
  Object.fromEntries(filterColumns.map((column) => [column.key, true])),
);

const visibleFilterColumns = computed(() =>
  filterColumns.filter((column) => columnVisibility[column.key]),
);

const hiddenFilterColumns = computed(() =>
  filterColumns.filter((column) => !columnVisibility[column.key]),
);

const filterGridTemplate = computed(() => {
  if (!visibleFilterColumns.value.length) {
    return '1fr';
  }
  return visibleFilterColumns.value.map((column) => column.width).join(' ');
});

const filteredRecords = computed(() => {
  const start = filters.startDate ? new Date(filters.startDate) : null;
  const end = filters.endDate ? new Date(filters.endDate) : null;
  const agency = filters.agency.trim();
  const keyword = filters.keyword.trim();

  return normalizedRecords.value.filter((record) => {
    const announcement =
      record.announcementDate !== '-'
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

const searchedRecords = computed(() => {
  const keyword = resultSearch.value.trim();
  if (keyword.length < 2) {
    return filteredRecords.value;
  }
  return filteredRecords.value.filter((record) => {
    return (
      record.title.includes(keyword) ||
      record.agency.includes(keyword) ||
      record.company.includes(keyword) ||
      record.id.includes(keyword)
    );
  });
});

const totalPages = computed(() => {
  const size = Math.max(1, Number(filters.numOfRows) || 1);
  return Math.max(1, Math.ceil(searchedRecords.value.length / size));
});

const pagedRecords = computed(() => {
  const size = Math.max(1, Number(filters.numOfRows) || 1);
  const page = Math.min(Math.max(1, filters.pageNo), totalPages.value);
  const startIndex = (page - 1) * size;
  return searchedRecords.value.slice(startIndex, startIndex + size);
});

const companySummaries = computed(() => {
  const summaryMap = new Map();

  filteredRecords.value.forEach((record) => {
    const entry = summaryMap.get(record.company) || {
      company: record.company,
      ceo: record.companyCeo,
      tel: record.companyTel,
      address: record.companyAddress,
      count: 0,
      agencies: new Map(),
    };
    if (entry.ceo === '-' && record.companyCeo !== '-') {
      entry.ceo = record.companyCeo;
    }
    if (entry.tel === '-' && record.companyTel !== '-') {
      entry.tel = record.companyTel;
    }
    if (entry.address === '-' && record.companyAddress !== '-') {
      entry.address = record.companyAddress;
    }
    entry.count += 1;
    entry.agencies.set(
      record.agency,
      (entry.agencies.get(record.agency) || 0) + 1,
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
        ceo: entry.ceo,
        tel: entry.tel,
        address: entry.address,
        count: entry.count,
        topAgencies,
      };
    })
    .sort((a, b) => b.count - a.count);
});

watch(
  () => [searchedRecords.value.length, filters.numOfRows, resultSearch.value],
  () => {
    const maxPage = totalPages.value;
    if (filters.pageNo > maxPage) {
      filters.pageNo = maxPage;
    }
    if (filters.pageNo < 1) {
      filters.pageNo = 1;
    }
  },
);

const parseLocalDate = (value) => {
  if (!value) return null;
  const [year, month, day] = value.split('-').map(Number);
  if (!year || !month || !day) return null;
  return new Date(year, month - 1, day);
};

const canSearch = computed(
  () => filters.startDate.trim() && filters.endDate.trim(),
);

const exceedsMaxRange = computed(() => {
  const start = parseLocalDate(filters.startDate);
  const end = parseLocalDate(filters.endDate);
  if (!start || !end) return false;
  const diffDays = Math.floor((end - start) / (1000 * 60 * 60 * 24));
  return diffDays > 365 * 5;
});

const validateDateRange = () => {
  const start = parseLocalDate(filters.startDate);
  const end = parseLocalDate(filters.endDate);
  if (!start || !end) {
    return '조회 기간을 올바르게 입력해 주세요.';
  }
  if (end < start) {
    return '종료일이 시작일보다 이전입니다.';
  }

  const todayLocal = new Date();
  todayLocal.setHours(0, 0, 0, 0);
  if (end > todayLocal) {
    return '조회 기간은 오늘 이전 날짜만 가능합니다.';
  }

  const diffDays = Math.floor((end - start) / (1000 * 60 * 60 * 24));
  if (diffDays > 365 * 5) {
    return '조회 기간은 최대 5년까지 가능합니다.';
  }

  return '';
};

const extractItems = (payload) => {
  const rawItems = payload?.items || [];
  return Array.isArray(rawItems) ? rawItems : [rawItems];
};

const buildQuery = (start, end) => {
  const params = new URLSearchParams({
    inqryBgnDt: buildDateTimeFromDate(start, '0000'),
    inqryEndDt: buildDateTimeFromDate(end, '2359'),
  });

  if (filters.agency.trim()) {
    params.set('dminsttNm', filters.agency.trim());
  }

  if (filters.keyword.trim()) {
    params.set('bidNtceNm', filters.keyword.trim());
  }

  return params.toString();
};

const fetchRecords = async () => {
  if (isLoading.value) {
    return;
  }
  if (!canSearch.value) {
    errorMessage.value = '조회 기간을 입력해 주세요.';
    return;
  }

  const validationMessage = validateDateRange();
  if (validationMessage) {
    errorMessage.value = validationMessage;
    return;
  }

  isLoading.value = true;
  errorMessage.value = '';
  progress.value = { current: 0, total: 1 };

  try {
    const start = parseLocalDate(filters.startDate);
    const end = parseLocalDate(filters.endDate);
    const query = buildQuery(start, end);
    const response = await fetch(`${endpoint}?${query}`, {
      method: 'GET',
    });
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    const result = await response.json();
    if (result?.resultCode && result.resultCode !== '00') {
      throw new Error(result.resultMsg || 'API error');
    }

    apiRecords.value = extractItems(result);
    totalCount.value = Number(result?.totalCount || apiRecords.value.length);
    filters.pageNo = 1;
    progress.value.current = 1;
  } catch (error) {
    errorMessage.value = `Fetch failed: ${error.message || 'Network error'}`;
    apiRecords.value = [];
    totalCount.value = 0;
  } finally {
    isLoading.value = false;
  }
};

const formatAmount = (value) => new Intl.NumberFormat('ko-KR').format(value);

const formatFilterCell = (record, column) => {
  if (column.key === 'amount') {
    return `${formatAmount(record.amount)}원`;
  }
  return record[column.key] ?? '-';
};

const formatTopAgency = (summary, index) => {
  const item = summary.topAgencies[index];
  return item ? `${item.name} (${item.count}건)` : '-';
};

const exportCompanySummaries = () => {
  const rows = companySummaries.value.map((summary) => ({
    '업체명': summary.company,
    '대표자명': summary.ceo,
    '업체전화번호': summary.tel,
    '업체주소지': summary.address,
    '낙찰건수': summary.count,
    'TOP1 수요기관': formatTopAgency(summary, 0),
    'TOP2 수요기관': formatTopAgency(summary, 1),
    'TOP3 수요기관': formatTopAgency(summary, 2),
  }));

  const headers = Object.keys(rows[0] || {});
  const escapeCsv = (value) => {
    const text = String(value ?? '');
    if (/[",\n]/.test(text)) {
      return `"${text.replace(/"/g, '""')}"`;
    }
    return text;
  };
  const csvLines = [
    headers.join(','),
    ...rows.map((row) => headers.map((key) => escapeCsv(row[key])).join(',')),
  ];
  const csv = `\uFEFF${csvLines.join('\n')}`;

  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');

  const today = new Date();
  const y = today.getFullYear();
  const m = String(today.getMonth() + 1).padStart(2, '0');
  const d = String(today.getDate()).padStart(2, '0');
  link.href = url;
  link.download = `업체별_낙찰_집계_${y}${m}${d}.csv`;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
};
</script>

<template>
  <div class="page">
    <header class="hero">
      <div class="hero__text">
        <p class="hero__tag">나라장터 낙찰 기록 인사이트</p>
        <h1>입찰 결과 기반 낙찰 업체 분석</h1>
        <p class="hero__desc">
          - 공고 게시기간, 수요기관, 키워드으로 낙찰 기록을 조회합니다.
        </p>
        <p class="hero__desc">
          - 필터링 조건 내에서 업체별 낙찰 건수와 주요 수요기관 TOP3를 한눈에
          확인할 수 있습니다.
        </p>
      </div>
    </header>

    <section class="filters">
      <div class="filters__header">
        <h2>조회 조건</h2>
        <p>조회 기간과 수요기관, 키워드를 입력해 검색하세요.</p>
      </div>
      <div class="filters__grid">
        <label class="input" :class="{ 'input--error': exceedsMaxRange }">
          <span>시작일</span>
          <input type="date" v-model="filters.startDate" />
        </label>
        <label class="input" :class="{ 'input--error': exceedsMaxRange }">
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
          <span>입찰공고명</span>
          <input
            type="text"
            v-model="filters.keyword"
            placeholder="입찰공고명"
          />
        </label>
      </div>
      <div class="filters__actions">
        <button
          class="button"
          type="button"
          :disabled="isLoading || exceedsMaxRange"
          @click="fetchRecords"
        >
          {{ isLoading ? '조회 중...' : '조회하기' }}
        </button>
        <div class="progress" v-if="isLoading && progress.total">
          <div class="progress__bar">
            <span
              class="progress__fill"
              :style="{
                width: `${Math.round((progress.current / progress.total) * 100)}%`,
              }"
            ></span>
          </div>
          <span class="progress__text">
            {{ Math.round((progress.current / progress.total) * 100) }}% ({{
              progress.current
            }}/{{ progress.total }})
          </span>
        </div>
        <p class="status" v-if="totalCount">API 총 {{ totalCount }}건</p>
        <p class="status status--error" v-if="exceedsMaxRange">
          조회 기간은 최대 5년까지 가능합니다.
        </p>
        <p class="status status--error" v-if="errorMessage">
          {{ errorMessage }}
        </p>
      </div>
    </section>

    <section class="content">
      <div class="panel">
        <div class="panel__header panel__header--split">
          <div class="panel__title-shift">
            <h2>필터링 결과</h2>
            <p>상단의 조회 조건에 따른 낙찰결과들 입니다.</p>
          </div>
          <label class="result-search result-search--inline">
            <span>키워드 검색</span>
            <input
              type="text"
              v-model="resultSearch"
              placeholder="2글자 이상 입력"
            />
          </label>
        </div>
        <div class="table table--filter">
          <div class="table__head">
            <div v-if="hiddenFilterColumns.length" class="table__head-hidden">
              <span class="table__head-hidden-label">숨김 컬럼</span>
              <label
                v-for="column in hiddenFilterColumns"
                :key="column.key"
                class="column-toggle"
              >
                <input type="checkbox" v-model="columnVisibility[column.key]" />
                <span>{{ column.label }}</span>
              </label>
            </div>
            <div
              class="table__head-row"
              :style="{ gridTemplateColumns: filterGridTemplate }"
            >
              <span v-for="column in visibleFilterColumns" :key="column.key">
                <label class="column-toggle">
                  <input
                    type="checkbox"
                    v-model="columnVisibility[column.key]"
                  />
                  <span>{{ column.label }}</span>
                </label>
              </span>
            </div>
          </div>
          <div class="table__body">
            <div v-if="!visibleFilterColumns.length" class="table__empty">
              표시할 컬럼이 없습니다. 상단에서 컬럼을 선택하세요.
            </div>
            <template v-else>
              <div
                v-for="record in pagedRecords"
                :key="record.id"
                class="table__row"
                :style="{ gridTemplateColumns: filterGridTemplate }"
              >
                <span
                  v-for="column in visibleFilterColumns"
                  :key="column.key"
                  :class="{ title: column.key === 'title' }"
                >
                  {{ formatFilterCell(record, column) }}
                </span>
              </div>
              <div v-if="isLoading" class="table__empty">조회 중입니다.</div>
              <div
                v-else-if="searchedRecords.length === 0"
                class="table__empty"
              >
                조건에 맞는 낙찰 기록이 없습니다.
              </div>
            </template>
          </div>
        </div>
        <div class="pagination" v-if="searchedRecords.length">
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
          <div class="panel__header panel__header--split panel__header--split-right">
            <div>
              <h2>업체별 낙찰 집계</h2>
              <p>필터링 결과를 기준으로 업체별 낙찰 건수와 주요 수요기관 TOP3</p>
            </div>
            <button
              class="button button--ghost button--success"
              type="button"
              :disabled="companySummaries.length === 0"
              @click="exportCompanySummaries"
            >
              CSV 다운로드
            </button>
          </div>
          <div class="table table--summary">
            <div class="table__head">
              <span>업체명</span>
              <span>대표자명</span>
              <span>업체전화번호</span>
              <span>업체주소지</span>
              <span>낙찰건수</span>
              <span>TOP1 수요기관</span>
              <span>TOP2 수요기관</span>
              <span>TOP3 수요기관</span>
            </div>
            <div class="table__body">
              <div
                v-for="summary in companySummaries"
                :key="summary.company"
                class="table__row"
              >
                <span class="title">{{ summary.company }}</span>
                <span>{{ summary.ceo }}</span>
                <span>{{ summary.tel }}</span>
                <span>{{ summary.address }}</span>
                <span>{{ summary.count }}건</span>
                <span>{{ formatTopAgency(summary, 0) }}</span>
                <span>{{ formatTopAgency(summary, 1) }}</span>
                <span>{{ formatTopAgency(summary, 2) }}</span>
              </div>
              <div v-if="isLoading" class="table__empty">집계 중입니다.</div>
              <div
                v-else-if="companySummaries.length === 0"
                class="table__empty"
              >
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
  font-family: 'Pretendard', 'SUIT', 'Apple SD Gothic Neo', sans-serif;
  background: radial-gradient(
    circle at top,
    #f6f0df 0%,
    #f2f5f8 45%,
    #ebeff3 100%
  );
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
  grid-template-columns: minmax(0, 1fr);
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

.result-search {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 12px;
  color: #4b4f55;
}

.result-search input {
  padding: 6px 12px;
  border-radius: 10px;
  border: 1px solid #d7cbbd;
  background: #fffdf9;
  font-size: 13px;
  outline: none;
  transition:
    border 0.2s ease,
    box-shadow 0.2s ease;
}

.result-search input:focus {
  border-color: #1d2026;
  box-shadow: 0 0 0 3px rgba(38, 50, 68, 0.15);
}

.result-search--inline {
  flex: 0 0 420px;
  align-items: center;
  flex-direction: row;
  gap: 10px;
  margin-left: 24px;
}

.result-search--inline input {
  flex: 1;
  min-width: 280px;
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
  transition:
    border 0.2s ease,
    box-shadow 0.2s ease;
}

.input input:focus {
  border-color: #1d2026;
  box-shadow: 0 0 0 3px rgba(38, 50, 68, 0.15);
}

.input--error input {
  border-color: #b1332b;
  box-shadow: 0 0 0 3px rgba(177, 51, 43, 0.15);
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
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease;
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

.button--success {
  color: #1b7f3a;
  border-color: #1b7f3a;
  background: #e9f7ee;
  border-width: 2px;
  font-weight: 700;
}

.button--success:not(:disabled):hover {
  box-shadow: 0 8px 16px rgba(27, 127, 58, 0.18);
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

.panel__header--split {
  display: flex;
  align-items: flex-start;
  justify-content: flex-start;
  gap: 24px;
  flex-wrap: wrap;
}

.panel__header--split-right {
  justify-content: space-between;
}

.panel__title-shift {
  margin-left: 12px;
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
  grid-template-columns: 1.2fr 0.9fr 0.9fr 1.7fr 0.6fr 1fr 1fr 1fr;
}

.table--summary .table__row span:nth-child(4) {
  word-break: break-word;
}

.table--filter .table__head {
  display: block;
  gap: 8px;
  text-transform: none;
  letter-spacing: normal;
  font-size: inherit;
  color: inherit;
  margin-top: 0px;
}

.table--filter .table__head-row {
  display: grid;
  gap: 12px;
  align-items: center;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-size: 11px;
  color: #7b7f86;
}

.table__head-hidden {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px 14px;
  font-size: 12px;
  color: #7b7f86;
}

.table__head-hidden-label {
  font-weight: 600;
  color: #1d2026;
}

.column-toggle {
  display: inline-flex;
  align-items: center;
  gap: 6px;
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

  .table--summary .table__head,
  .table--summary .table__row {
    grid-template-columns: 1.4fr 0.8fr 1.2fr;
  }

  .table--summary .table__row span:nth-child(2),
  .table--summary .table__row span:nth-child(3),
  .table--summary .table__row span:nth-child(4),
  .table--summary .table__row span:nth-child(7),
  .table--summary .table__row span:nth-child(8),
  .table--summary .table__head span:nth-child(2),
  .table--summary .table__head span:nth-child(3),
  .table--summary .table__head span:nth-child(4),
  .table--summary .table__head span:nth-child(7),
  .table--summary .table__head span:nth-child(8) {
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
