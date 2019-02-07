# 뉴스데이터로 주가지수 예측하는 논문 재현하기

나는 개인적으로 주식과 주가데이터에 관심이 많다. <br>
<br>
이 글은 내가 예전에 [뉴스와 주가 빅데이터 감성분석을 통한 지능형 의사결정모형](http://www.dbpia.co.kr/Journal/ArticleDetail/NODE01901732) 이라는 논문을 읽고 직접 재현해본 기록을 담은 글이다. <br>
<br>
결론부터 말하자면, 실패했다(내 경우엔 실패했다. 다른 재야의 고수들은 어떨지 모르겠다). <br>
이 글은 나의 실패를 기록한 글이다. <br>
<br>
<br>

### Motive

해당 논문을 간단하게 요약하면 이렇다. <br>
<br>
1. 경제관련 인터넷 뉴스는 그날 주가지수의 등락과 연관성이 있을것이다.<br>
2. 경제관련 인터넷 뉴스를 형태소 분석기에 집어넣어서 쓰인 단어들의 빈도를 뽑아낸다.<br>
3. 주가지수가 오른날에 쓰인 단어들엔 점수를 주고, 내린날에 쓰인 단어들엔 점수를 깎는다. 이를 '각 단어의 감성을 분석한다' 라고 부른다.<br>
4. 이런 방식으로 오랜 기간동안의 뉴스 데이터를 분석한다면 주가지수의 등락과 높은 연관성을 지닌 단어들을 발견할 수 있을것이다.<br>
<br>
나의 실험도 위와 같은 목차로 진행된다.<br>
<br>
<br>

### 뉴스 수집

뉴스 데이터 수집엔 파이썬의 [Scrapy][https://scrapy.org/] 라는 라이브러리를 사용했다. <br>

수집 대상으로 결정한 뉴스 포털은 네이버 금융의 [뉴스포커스 - 시황 및 전망](https://finance.naver.com/news/news_list.nhn?mode=LSS3D&section_id=101&section_id2=258&section_id3=401) 게시판이다.
2005년 1월 1일부터 2017년 12월 1일까지 대략 13년간의 뉴스 데이터를 긁어왔다.

'시황 및 전망' 카테고리 말고도 다른 카테고리에서도 긁어오려고 했으나, 다른 카테고리의 경우 

단순히 현재 어떤 수치에 대한 정보만 알려주고 끝나는 경우도 있고(이런 경우는 기사의 내용도 한줄 또는 두줄밖에 안되는 경우가 많아 너무짧다)
*ex) "ㅇㅇㅇ 지수 1.2포인트 상승 출발", "ㅇㅇㅇ 지수 2.6포인트 하락 마감", "ㅁㅁㅁ 기업 영업이익 작년도 대비 12.7% 증가*

숫자가 너무 많이 들어간 경우도 있고 (숫자는 감성분석을 할 수 없다)
*ex) "지수선물 하락..286.30(-0.65p)"*

기업명이 들어간 기사가 너무 많이 올라오는 경우도 있는등
*ex) "ㅇㅇ제약, 작년 영업익 전년比 29.5%↑", "ㅁㅁ바이오, 상한가 진입.. +29.90% ↑"*


올라오는 뉴스의 퀄리티를 살펴봤을때 '시황 및 전망' 카테고리가 가장 괜찮은듯 하여 선택했다. 여기도 문제가 없는건 아닌데 이중에서 그나마 제일 낫다.
3번째 예시로든 '기업명이 들어간 기사'가 왜 안되냐면, [온라인 언급이 기업 성과에 미치는 영향 분석](http://www.dbpia.co.kr/Journal/ArticleDetail/NODE06596994?TotalCount=722&Seq=13&q=(%5B%EA%B9%80%EB%8F%99%EC%84%B1%C2%A7coldb%C2%A72%C2%A751%C2%A73%5D)&searchWord=%EC%A0%84%EC%B2%B4%3D%5E%24%EA%B9%80%EB%8F%99%EC%84%B1%5E*&searchWordCondition=%EC%9E%90%EB%A3%8C%EC%9C%A0%ED%98%95%3D%5E%24%EC%A0%84%EC%B2%B4%5E*&Multimedia=0&isIdentifyAuthor=0&Collection=0&isFullText=0&specificParam=0&SearchMethod=0&Sort=1&SortType=desc&Page=1&PageSize=20) 에 따르면 
