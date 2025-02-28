---
layout: page
title: "검색"
permalink: /search/
---

<div id="search-container">
  <input type="text" id="search-input" placeholder="검색어 입력..." class="search-input" />
  <div id="results-container"></div>
</div>

<style>
  .search-input {
    padding: 8px;
    width: 100%;
    max-width: 400px;
    margin-bottom: 10px;
    font-size: 16px;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
  
  #results-container {
    margin-top: 20px;
  }
  
  .result-item {
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 1px solid #eee;
  }
  
  .result-title {
    font-size: 18px;
    font-weight: bold;
    margin-bottom: 5px;
  }
  
  .result-date {
    color: #777;
    font-size: 14px;
    margin-bottom: 8px;
  }
  
  .result-snippet {
    font-size: 14px;
    color: #333;
  }
  
  .highlight {
    background-color: yellow;
    font-weight: bold;
  }
</style>

<script src="https://cdnjs.cloudflare.com/ajax/libs/lunr.js/2.3.9/lunr.min.js"></script>
<script>
document.addEventListener('DOMContentLoaded', function() {
  // 검색 인덱스 및 문서 저장소
  let idx;
  let documents = {};
  
  // DOM 요소
  const searchInput = document.getElementById('search-input');
  const resultsContainer = document.getElementById('results-container');
  
  // search.json 로드
  fetch('{{ site.baseurl }}/search.json')
    .then(response => response.json())
    .then(data => {
      // 문서 저장소 구성
      data.docs.forEach(doc => {
        documents[doc.url] = doc;
      });
      
      // lunr 인덱스 구성 - 부분 검색 지원 설정
      idx = lunr(function() {
        // 기본 설정
        this.ref('url');
        this.field('title', { boost: 10 });
        this.field('content');
        
        // 토큰화 파이프라인 수정 - 부분 단어 검색을 위한 설정
        this.pipeline.remove(lunr.stemmer);
        this.searchPipeline.remove(lunr.stemmer);
        
        // n-gram 토큰화를 위한 함수 추가
        this.tokenizer.separator = /[\s\-]+/;
        
        // 문서 추가
        data.docs.forEach(doc => {
          this.add({
            'url': doc.url,
            'title': doc.title,
            'content': doc.content,
            'date': doc.date
          });
        });
      });
      
      // 검색 입력란 활성화
      searchInput.disabled = false;
      searchInput.placeholder = "검색어를 입력하세요";
    })
    .catch(error => {
      console.error('검색 인덱스를 로드하는 중 오류가 발생했습니다:', error);
      resultsContainer.innerHTML = '<p>검색 데이터를 로드할 수 없습니다.</p>';
    });
  
  // 검색 이벤트 리스너
  searchInput.addEventListener('input', function() {
    const query = this.value.trim();
    
    if (query.length < 1) {
      resultsContainer.innerHTML = '';
      return;
    }
    
    if (!idx) {
      resultsContainer.innerHTML = '<p>검색 인덱스를 로드 중입니다...</p>';
      return;
    }
    
    // 부분 검색을 위해 각 단어에 와일드카드 추가
    const wildcardQuery = query.split(' ')
      .map(term => term + '*')
      .join(' ');
    
    // 여러 검색 쿼리 형식 조합 (더 많은 결과를 얻기 위해)
    const results = idx.search(wildcardQuery);
    
    // 부분 검색으로 결과가 충분하지 않으면 퍼지 검색 추가
    if (results.length < 3 && query.length > 2) {
      const fuzzyQuery = query.split(' ')
        .map(term => term + '~1')
        .join(' ');
      
      const fuzzyResults = idx.search(fuzzyQuery);
      
      // 결과 병합
      fuzzyResults.forEach(result => {
        if (!results.some(r => r.ref === result.ref)) {
          results.push(result);
        }
      });
    }
    
    if (results.length === 0) {
      // 결과가 없으면 제목과 내용에서 직접 부분 문자열 검색 (Lunr 우회)
      const matchedDocs = Object.values(documents).filter(doc => {
        return doc.title.toLowerCase().includes(query.toLowerCase()) || 
               doc.content.toLowerCase().includes(query.toLowerCase());
      });
      
      if (matchedDocs.length > 0) {
        let resultHtml = '';
        matchedDocs.forEach(doc => {
          // 내용 일부 추출 (약 150자)
          let snippet = doc.content.length > 150 
            ? doc.content.substring(0, 150) + '...' 
            : doc.content;
            
          // 검색어 하이라이트
          const titleHighlighted = highlightText(doc.title, query);
          const snippetHighlighted = highlightText(snippet, query);
          
          resultHtml += `
            <div class="result-item">
              <div class="result-title">
                <a href="${doc.url}">${titleHighlighted}</a>
              </div>
              <div class="result-date">${doc.date}</div>
              <div class="result-snippet">${snippetHighlighted}</div>
            </div>
          `;
        });
        
        resultsContainer.innerHTML = resultHtml;
      } else {
        resultsContainer.innerHTML = '<p>검색 결과가 없습니다.</p>';
      }
      
      return;
    }
    
    // 결과 표시
    let resultHtml = '';
    results.forEach(result => {
      const doc = documents[result.ref];
      
      // 내용 일부 추출 (약 150자)
      let snippet = '';
      
      // 검색어가 포함된 부분 찾기
      const lowerContent = doc.content.toLowerCase();
      const lowerQuery = query.toLowerCase();
      const index = lowerContent.indexOf(lowerQuery);
      
      if (index !== -1) {
        // 검색어 주변 텍스트 추출
        const start = Math.max(0, index - 60);
        const end = Math.min(doc.content.length, index + query.length + 60);
        snippet = doc.content.substring(start, end);
        
        // 시작이나 끝이 잘렸다면 표시
        if (start > 0) snippet = '...' + snippet;
        if (end < doc.content.length) snippet += '...';
      } else {
        // 검색어가 없으면 처음 150자
        snippet = doc.content.length > 150 
          ? doc.content.substring(0, 150) + '...' 
          : doc.content;
      }
      
      // 검색어 하이라이트
      const titleHighlighted = highlightText(doc.title, query);
      const snippetHighlighted = highlightText(snippet, query);
      
      resultHtml += `
        <div class="result-item">
          <div class="result-title">
            <a href="${doc.url}">${titleHighlighted}</a>
          </div>
          <div class="result-date">${doc.date}</div>
          <div class="result-snippet">${snippetHighlighted}</div>
        </div>
      `;
    });
    
    resultsContainer.innerHTML = resultHtml;
  });
  
  // 검색어 하이라이트 함수
  function highlightText(text, query) {
    if (!query || query.trim() === '') return text;
    
    const words = query.trim().toLowerCase().split(/\s+/);
    let result = text;
    
    words.forEach(word => {
      if (word.length < 2) return; // 너무 짧은 단어는 건너뜀
      
      const regex = new RegExp('(' + word + ')', 'gi');
      result = result.replace(regex, '<span class="highlight">$1</span>');
    });
    
    return result;
  }
  
  // 검색 입력란 초기 상태
  searchInput.disabled = true;
  searchInput.placeholder = "검색 인덱스 로딩 중...";
});
</script>