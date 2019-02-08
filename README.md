# 뉴스데이터로 주가지수 예측하는 논문 재현하기

나는 개인적으로 주식과 주가데이터에 관심이 많다. <br>
<br>
이 글은 내가 예전에 [뉴스와 주가 빅데이터 감성분석을 통한 지능형 의사결정모형](http://www.dbpia.co.kr/Journal/ArticleDetail/NODE01901732) 이라는 논문을 읽고 직접 재현해본 기록을 담은 글이다. <br>
<br>
결론부터 말하자면, 실패했다(내 경우엔 실패했다. 다른 재야의 고수들은 어떨지 모르겠다).

이 글은 그 과정을 기록한 글이다.

<br>
<br>

### Motive

해당 논문을 간단하게 요약하면 이렇다. 

1. 경제관련 인터넷 뉴스는 그날 주가지수의 등락과 연관성이 있을것이다.<br>
2. 경제관련 인터넷 뉴스를 형태소 분석기에 집어넣어서 해당 뉴스에 쓰인 단어들의 빈도를 뽑아낸다.<br>
3. 주가지수가 오른날의 뉴스기사에 쓰인 단어들엔 점수를 주고, 내린날에 쓰인 단어들엔 점수를 깎는다. 이를 '단어의 감성을 분석한다' 라고 부른다.<br>
4. 이런 방식으로 오랜 기간동안의 뉴스 데이터를 분석한다면 주가지수의 등락과 높은 연관성을 지닌 단어들을 발견할 수 있을것이다.<br>
5. 이 단어들로 주가지수를 예측해보자. <br>
<br>
나의 실험도 위와 같은 목차로 진행된다.<br>
<br>
<br>

### 뉴스 수집

뉴스 데이터 수집엔 파이썬의 [Scrapy](https://scrapy.org/) 라는 라이브러리를 사용했다.

수집 대상으로 결정한 뉴스 포털은 네이버 금융의 [뉴스포커스 - 시황 및 전망](https://finance.naver.com/news/news_list.nhn?mode=LSS3D&section_id=101&section_id2=258&section_id3=401) 게시판이다.

![naver_financial_news](https://user-images.githubusercontent.com/47002080/52473231-9be64300-2bd8-11e9-92cb-6e9e597f6a66.PNG)

2005년 1월 1일부터 2017년 12월 1일까지 대략 13년간의 뉴스 데이터를 긁어왔다.

'시황 및 전망' 카테고리 말고도 다른 카테고리에서도 긁어오려고 했으나, 다른 카테고리의 경우 

단순히 현재 어떤 수치에 대한 정보만 알려주고 끝나는 경우도 있고(이런 경우는 기사의 내용도 한줄 또는 두줄밖에 안되는 경우가 많아 너무짧다) <br>
*ex) "ㅇㅇㅇ 지수 1.2포인트 상승 출발", "ㅇㅇㅇ 지수 2.6포인트 하락 마감", "ㅁㅁㅁ 기업 영업이익 작년도 대비 12.7% 증가*

숫자가 너무 많이 들어간 경우도 있고 (숫자는 감성분석을 할 수 없다) <br>
*ex) "지수선물 하락..286.30(-0.65p)"*

기업명이 들어간 기사가 너무 많이 올라오는 경우도 있는등 <br>
*ex) "ㅇㅇ제약, 작년 영업익 전년比 29.5%↑", "ㅁㅁ바이오, 상한가 진입.. +29.90% ↑"*

올라오는 뉴스의 퀄리티를 봤을때 '시황 및 전망' 카테고리가 가장 괜찮은듯 하여 선택했다. 여기도 문제가 없는건 아닌데 이중에서 그나마 제일 낫다.

3번째 예시로든 '기업명이 들어간 기사'를 제외시키기로 결정한 이유는, 논문의 본문에 '기업의 주가가 하락세로 돌아서면 기업에서는 홍보성 뉴스기사를 뿌리기 때문에(!) 각 종목에 대한 개별 기사의 경우는 정확도가 떨어진다' 라는 내용을 참고했기 때문이다. 

또한 [온라인 언급이 기업 성과에 미치는 영향 분석](http://bit.ly/2RKCLc3) 라는 논문에 따르면, "기업명이 사용된 뉴스의 경우, 기업의 실제 경영상태와 관련이 있는 뉴스인지 아니면 그냥 단순 언급인지 구분하지 않는한 정확도가 매우 떨어진다" 라고 한다. 그래서 뺐다.

즉, 예를들면 경제 칼럼니스트가 쓴 글처럼 내용이 길고 경제의 동향과 관련된 단어가 많이 쓰인 뉴스만 수집하는것에 중점을 뒀다.

이렇게 하여 수집된 뉴스데이터들

![collected_news_data](https://user-images.githubusercontent.com/47002080/52473223-9ab51600-2bd8-11e9-895a-c82ea189781c.png)

<br>
<br>

### 단어 추출

###### 형태소 분석기

이제 수집한 뉴스를 형태소 분석기로 분석할 차례다. 

여기엔 여러가지 형태소 분석기 라이브러리들을 모아서 똑같은 인터페이스로 사용할수 있게 해주는 [KoNlpy](https://konlpy-ko.readthedocs.io/ko/v0.4.3/) 라는 라이브러리를 사용했다.

형태소 분석기에는 트위터, 꼬꼬마, 코모란 등등 여러가지가 있지만 여기저기 구글링을 해봐도 대체로 Mecab이 독보적이라 이걸 사용한다. <br>
<br>

###### 필터링

Mecab에 13년치 모든 뉴스를 전부 집어넣어서 뽑아낸 단어셋(set)을 보니, 전체 뉴스 데이터(약 20만건)에서 딱 한건의 기사에만 나오거나 그에 준할만큼 적은 기사에만 등장하는 단어가 극단적으로 많았다. 이런 단어들은 실험에 노이즈만 발생시키고 설명력이 없을거라 생각되기 때문에 분석에 쓰일 단어는 최소 50건 이상의 기사에 등장하는 단어로 한정한다.

한음절 단어와 명사를 제외한 모든 품사를 제거하였고 기업명(상장폐지포함)과 사람이름(ex: 기자이름)또한 제거했다. 

또한 '무단배포' '전재금지' '기자' 이런 단어들은 쓸모도 없는데 거의 모든 기사에서 나타나기 때문에 빈도수를 기준으로 한 백분위로 상/하위 3%의 단어는 쳐낸다.

이로써 분석에 쓰일 가장 기본적인 단어셋이 만들어졌다.

![wordset_basic_filtered](https://user-images.githubusercontent.com/47002080/52473236-9d177000-2bd8-11e9-9c78-38f320c81e90.PNG)

참고로 '전체 뉴스문서중 해당 단어가 나온 문서의 갯수' 를 DF(Document Frequency) 라고 부른다. 위의 '무단배포' '전재금지' 등의 단어는 DF가 과도하게 높을것이다. DF는 내가 만든건 아니고 검색기술과 관련된 업계에서 쓰이는 용어이다.

이런 용어중엔 TF(Term Frequency) 라는것도 있다. TF란 어떤 한개의 문서내에서 해당 단어가 얼마나 자주 등장하는지를 센 숫자이다. 예를들어 라면을 끓이는 방법에 대한 문서에서는 '라면'의 TF가 높을것은 자명하다. 

DF와 TF는 이제 곧 이어질 내용과 밀접한 연관이 있기 때문에 한번 짚고 넘어간다.

<br>
<br>

### 감성 사전

이제 각 단어의 감성이 긍정(이 단어가 쓰인 날엔 주가지수가 오른다)인지 또는 부정인지를 분석할 차례다. 단어별로 점수를 매기게 되는데 이를 '감성점수'라고 부르겠다. 점수가 클수록 강한 긍정이며 작을수록 강한 부정이다. 

각 단어별로 감성점수를 매겨놓은 집합을 '감성사전'이라고 부른다. <br>
<br>

###### 감성점수 공식

단어의 감성점수를 매기는 방법은 논문마다 조금씩 다르긴 하지만 대체로 비슷한 모습을 띈다. 여러 논문들을 참고하여 만든 나의 단어별 감성점수 공식은 다음과 같다 

![formular_word](https://user-images.githubusercontent.com/47002080/52473230-9be64300-2bd8-11e9-9cc7-94b2e745d145.png)

예를들어, 코스피의 등락이 30p, -10p, -5p 인 날짜에 각각 올라온 뉴스에서 단어가 쓰였다면 이 단어의 감성점수는 (30-10-5)/3 = 5가 된다. 공식의 핵심컨셉은 "상승장에 올라온 기사에 많이 쓰인 단어일수록 점수가 높다. 그리고 강한 상승장일수록 더 큰 점수를 준다." 이다. 하락장일 경우 역시 해당된다.

아쉽게도 위의 식은 TF에 대한 개념이 들어가 있지 않다. 예를들어 어떤날에 코스피가 50p나 올라서 상당한 강세장이라고 칠때, 이날 올라온 뉴스에 어떤 단어가 엄청 많이 사용되도 위 공식의 분모엔 +50밖에 안더해진다. 반대로 이날에 어떤 단어가 딱 한번밖에 안쓰였더라도, 일단 쓰였으면 이 역시 +50을 받는다.

TF에 해당하는 개념을 고려하지 않는점은 아쉽긴 하지만 일단은 진행하기로 결정했다. 만약 정말로 이 세상에 '뉴스와 주가지수간의 연관성'이 존재한다면, 이렇게 약간 덜 정확한 방식으로 얻어낸 결과물에서도 투박하게나마 경향성을 확인할수 있을것이라고 생각했기 때문이다. 일단 경향성을 확인했다면 본격적인 최적화는 나중에 천천히 해도 된다. <br>
<br>

###### TF-IDF

처음엔 이렇게 얻은 감성점수에다 해당 단어의 tf-idf 값도 곱했었다.

tf-idf 는 검색업계에 쓰이는 용어인데, 분자자리에 tf가 위치하고 분모자리에 df 위치한 값이다. 쉽게 말하면 "문서에서 자주 등장할수록(TF가 클수록), 그러나 전체 문서에서는 등장빈도가 낮을수록(DF가 작을수록) 커지는 값" 이며 검색을 할때 해당 단어의 중요도를 나타낸다고 한다.

어쨌든 이 tf-idf값을 곱했더니, 평균적으로 0.5근처에서 머물던 단어들의 감성점수가 모두 0.05 이하로 내려가 버렸다. 그래서 폐기했다.<br>
<br>

###### News score

이제 일련의 과정을 통해 만들어진 감성사전으로 뉴스를 분석할 차례다.

어떤 뉴스의 감성점수를 계산하는 공식은 다음과 같다.

![formular_news](https://user-images.githubusercontent.com/47002080/52473228-9be64300-2bd8-11e9-9c7b-596ba215c66d.png)

쉽게 말하면 해당뉴스에서 쓰인 모든 단어의 감성점수의 평균이다.<br>
<br>

###### Daily score

이렇게 얻어진 뉴스의 감성점수를 사용하여 해당 날짜의 감성점수를 계산한다.

공식은 다음과같다.

![formular_days](https://user-images.githubusercontent.com/47002080/52473227-9b4dac80-2bd8-11e9-8d95-bfadc08d7f42.png)

이 역시 그날 올라온 뉴스들의 감성점수를 평균낸것이다.

이렇게 얻어진 '어떤 날의 감성점수(이하 Daily_score)' 를 가지고 어떻게 주가지수를 예측한다는 것일까?<br>
<br>


###### 임계점

나는 '시황 및 전망' 게시판에서 뉴스를 수집할때 모든 뉴스를 한곳에 모아서 수집한게 아니라 언론사별로 나눠서 수집했다.

이는 [주가지수 예측을 위한 뉴스 빅데이터 오피니언마이닝 모형](http://m.riss.kr/search/detail/DetailView.do?p_mat_type=be54d9b8bc7cdb09&control_no=beda0ddd0c87e40dffe0bdc3ef48d419) 라는 논문에 나온 방식을 참고한것인데, 이 논문에서는 '각 언론사별로 논조가 다르듯이 현재 주식시장에 대한 스탠스도 다를것이다' 라고 가정했다.

이를테면 평소에 경제상황을 보수적으로 전망하는 언론사에서는 뉴스들의 감성점수가 대체로 낮게 나타날것이고 낙관적으로 전망하는 언론사에서는 뉴스들의 전체적인 감성점수가 높게 나타날것이다.

![different_senti_score_by_media](https://user-images.githubusercontent.com/47002080/52473224-9b4dac80-2bd8-11e9-8e96-6a80bc2fee54.PNG)

때문에 언론사별로 각기 다른 '임계점' 을 구한뒤 이를 바탕으로 주가지수를 예측한다는게 주 컨셉이다.

Daily_score가 임계점보다 높으면 '이날의 주가지수가 상승한다고 예측' 한것이며 낮으면 '이날의 주가지수가 하락한다고 예측' 한것이다.

임계점은 -5부터 5까지, 0.02단위로 순차적으로 올려간다. 500번의 반복문을 수행하는 것이다. -5부터 5라는 구체적인 범위로 잡은 이유는 Daily_score가 대체로 -2부터 2사이에 분포하는걸 확인했기 때문이다. 그래서 넉넉하게 -5와 5로 잡았다.

이와 같은 아이디어를 바탕으로 '최소 50개 이상의 문서에서 등장, 그리고 이걸 빈도수 백분율 상하위 3%를 쳐낸 단어셋' 으로 예측한 결과는 다음과 같다.

![result_basic](https://user-images.githubusercontent.com/47002080/52473232-9be64300-2bd8-11e9-86a6-fdf0a511c2a0.PNG)

빨간색 accuracy가 우리가 생각하는 그 정확도를 나타내며 나머지 recall과 precision 그리고 F1 score 지표는 내가 예전에 코세라에서 머신러닝 강의를 들으면서 배운 지표를 나타내봤다. 모두 높을수록 좋은것이며 이 지표들에 대한 자세한 설명은 생략한다.

정확도는 대충 69퍼센트정도 된다.

물론 감성사전을 만드는데 쓰인 데이터에 그대로 실험한 결과라 당연히 overfitting이 있겠지만, 13년에 가까운 상당한 데이터에 적용했음에도 70%에 가까운 정확도는 상당히 고무적이라 할수 있다. 논문에서 결과로 제시한 수치와도 얼추 들어맞는다. 이때까지만 해도 '와 미쳤네.. 1년에 시장이 열리는 날을 240일, 욕심부리지 말고 하루에 3%만 먹으며, 승률 70% 라고 가정하면 1년에 수익이 17배 라는건데? 백만원만 넣어놔도 다음해에 1700만원이 되다니' 하면서 행복회로가 돌아가기 시작했다.  

내가 이 프로젝트를 '실패했다'고 느낀건 최적화를 진행하면서 부터다. <br>
<br>
<br>


### 최적화

###### 쓸모없는 기사제목 제외

뉴스를 수집하다 보니 쓸모없는 기사들이 눈에 띄었다.

제목이 [부고] 또는 [인사] 로 시작하는 기사인데 예를들면 *"[부고]ㅇㅇ은행 ㅁㅁㅁ씨 모친상", "[인사]공정거래위원위 인사"* 같은 형식이다.

상식적으로 ㅇㅇ은행의 ㅁㅁㅁ씨가 모친상을 당했다는 기사는 주가지수 예측에 도움이 안될것이라 생각되기에 제외시켜야 한다. 또한 내가 만든 모델에서는 사람이름으로 감성분석을 하지 않기 때문에 기업명과 사람의 이름이 본문의 주된 컨텐츠가 되는 *[인사]* 기사들도 제외시켰다.

이처럼 쓸모없는 내용이 주된 뉴스기사 제목을 제외 시킨뒤 남은 뉴스들만 가지고 감성사전을 만든다. <br>
최적화를 진행한 '쓸모없는 뉴스기사의 제목패턴' 의 순서는 아래와 같다. <br>

1. (마감) (출발)
2. [인사] [부고]
3. 각종기업명들
4. 메모]
5. 메모] (마감) (출발) 표
6. 상승 하락 출발 마감

이에 따른 결과는 아래와 같다.

![results_of_trash_title_optimization](https://user-images.githubusercontent.com/47002080/52473234-9c7ed980-2bd8-11e9-9e07-faff51c90674.gif)

사진 상단에 까만색 영역이 있는 사진부터가 1번 최적화의 결과이다. 제외할 뉴스제목 패턴을 더하면 더할수록 accuracy를 비롯한 모든 지수들이 하락하는걸 볼수 있다. 중간에 한번 오르는것도 최적화가 잘되서 그런게 아니라 아마 내가 '정확도를 깎는' 최적화를 일단 제거하고 실행해서 그럴거다.

위의 패턴에 해당하는 뉴스를 직접 보면 알겠지만, 이 뉴스들은 상식적으로 '이 세상에 뉴스와 주가지수간의 연관성이 존재한다면' 그걸 밝히는데 전혀 쓸모가 없을것이 확실한 텍스트를 담고있다. 그럼에도 이를 제거할수록 기계적으로 정확도 하락이라는 결과가 나타나는 것이다.

문득 이런 생각이 들었다.

>초반에 아무 처리도 안한 데이터에서 결과가 잘나온 이유는 감성사전이 뉴스기사들에 overfitting 되었기 때문이며<br>
몇몇 뉴스기사를 제외한뒤 실행한 실험에서 결과가 안좋아진 이유는 그저 overfitting이 덜되었기 때문이 아니었을까.

*감성사전 제작에 쓰인 뉴스 데이터가 많다 -> 오버피팅된다 -> 결과가 좋다 <br>
뉴스데이터가 적다 -> 오버피팅이 안된다 -> 결과가 안좋다.*

이 말은 즉, "뉴스기사와 주가지수 사이에 연관성 따위는 없다, 적어도 이 모델로는 밝힐수 없다" 라는뜻 아닌가.

이런 의심은 또다른 최적화를 진행한뒤 확실해졌다. <br>
<br>

###### 상승/하락폭이 컸던 날들만

[뉴스 콘텐츠의 오피니언 마이닝을 통한 매체별 주가상승 예측정확도 비교연구](http://bit.ly/2UPp5OP) 논문에서는 뉴스데이터를 수집한뒤 이를 코스피의 등락폭이 가장컸던 상위 15%만 걸러낸다.

나도 이걸 보고 괜찮은 아이디어라 생각해서 일단은 코스피의 등락폭이 큰 순서대로 하위 50%는 쳐낸뒤 실험을 해봤다. 실제로 수집된 코스피지수를 보면 변동이 거의 없는(1포인트 미만) 날도 상당히 많기 때문에 이 최적화는 합리적으로 보인다. 이 모델로 뉴스와 주가지수간의 연관성을 밝힐수 있다면, 이 최적화로 인해 반드시 정확도가 올라야만 한다. 

![result_upperratio_50per](https://user-images.githubusercontent.com/47002080/52473233-9c7ed980-2bd8-11e9-82b7-921a61178686.PNG)

글의 초반부 '임계점' 항목에 맨 처음 제시된 그래프에 비해 좌우로 넒어졌고 높이는 낮아졌다.

나의 의심이 맞다면 이는 당연한 결과다. 정확도를 예측하는데에는 모든 뉴스데이터가 쓰이지만 감성사전을 만드는데는 절반(상위 50%)의 뉴스데이터밖에 쓰이지 않았다. 감성사전 입장에서는 나머지 50%의 뉴스데이터는 처음보는 애들이기 때문에, 즉 오버피팅이 덜 됬기 때문에 '반드시 올라야만 하는' 정확도는 오히려 떨어졌다.

평소보다 적은양의 데이터로 감성사전을 만들었으니 임계점이 좁혀질수록 감성사전은 평소보다 더 빨리, 민감하게 반응하기 시작한다. 그래서 그래프의 좌우는 넓은것이다.

무엇보다 결정적인 사실이 하나 있다. 나는 이 최적화를 진행할때 절반의 데이터로 감성사전을 만들었고, 주가지수 예측 역시 이 절반의 데이터로 진행해버린 실수를 한적있다. 감성사전 입장에서는 전부 자기가 아는 뉴스데이터만 만나게 된다. 규모만 작아졌을뿐 처음과 상황이 똑같은것이다. 아니, 오히려 오버피팅은 더 밀도있게 심화된다. 사용된 데이터가 원래의 절반이니까.

이런 실수를 해서 진행한 예측에서 주가지수 예측의 정확도는 무려 76.4%가 나왔다. (로그 자료는 잃어버렸다)

지금까지의 실험결과들을 종합해보면, 그저 무조건 오버피팅이 될수록 결과가 좋았고 오버피팅이 덜 될수록 결과가 나빠졌을 뿐이다. 이 모델로는 뉴스와 주가지수의 연관성을 예측할수 없다. <br>
<br>

###### 눈으로 확인

최적화는 아니지만 직접 Daily_score를 로깅해본적이 있었다. 그중 다른날에 비해 특히 daily_score가 높았던 날의 뉴스를 읽어봤다. 허나.. 딱히 경제상황을 낙관적으로 전망했다는 느낌은 전혀 받을 수 없었다.

오히려 경제상황을 긍정적으로 바라보는 뉴스기사를 읽어본뒤 '이런 기사가 올라올정도면 이 날의 daily_score는 높겠는데?' 하고 로깅해봤더니 daily_score가 플러스도 아니고 -0.04인 경우도 있었다. 이거 말고도, 비교를 하면할수록 daily_score와 뉴스의 논조 사이에 연관성을 전혀 느낄수가 없었다.

물론 눈으로 모든 케이스를 전수조사 한건 아니지만 이런 경험들이 쌓이면서 이 실험 자체에 대한 나의 회의감은 커져만 갔다. <br>
<br>
<br>

### 마치며

여기까지 실험이 진행됬을때 사실상 나는 여기서 마음이 떠났기 때문에 프로젝트는 이렇게 흐지부지 끝나고 말았다.

원래는 각 언론사별로 그날 주가지수의 등락을 5단계로(강한하락, 하락, 변동없음, 상승, 강한상승) 예측한뒤 이를 머신러닝의 인자로 줘서 겸사겸사 머신러닝 공부까지 해보려고 했으나 지금와선 안될일이 되었다.

그러다 [뉴스 콘텐츠의 오피니언 마이닝을 통한 매체별 주가상승 예측정확도 비교연구](http://bit.ly/2UPp5OP) 를 보다가 이런 표가 눈에 띄었다.

![too_small_data](https://user-images.githubusercontent.com/47002080/52473235-9c7ed980-2bd8-11e9-8f05-36f7dcd38014.PNG)

후.. 40일 분량이면.. 오버피팅이 되다못해 아예 원하는 결과가 나올때까지 구간을 입맛대로 고를수 있을거 같은데. 하여튼 다 끝나고 나니 이런게 보인다.<br>
다른 논문들도 찾아봤는데 대체로 비슷하다. 대부분 30~40일 분량의 뉴스데이터만 사용했다. 글 도입부에 제시한 논문의 경우 766건의 뉴스데이터를 사용했다고 한다..

그리고 뜬금없지만 MongoDB의 단점을 발견하기도 했다. 수집한 뉴스데이터는 대략 17만건정도 되는데, 몽고DB에 insert 하면  500~1000건이 그냥 사라진다. 안그럴때도 있지만 사라질때는 정말 아무 이유없이 사라진다. 

때문에 민감한 내용의 빅데이터를 저장할때는 몽고디비를 쓰면 안되겠다. 이런 짤막한 팁 덕에 현재 주가데이터를 수집하는데 사용되는 데이터베이스는 Mysql과 Redis 이다.

![money_swag](https://user-images.githubusercontent.com/47002080/52473226-9b4dac80-2bd8-11e9-93a8-5d6e68fb256c.gif)

역시 편하게 앉아서 불로소득을 얻기란 쉽지 않다.
