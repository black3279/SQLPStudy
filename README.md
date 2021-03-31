# SQLPStudy

BCHR = ( 캐시에서 곧바로 찾은 블록 수 / 총 읽은 블록 수 ) * 100 =  ( ( 1 - 물리적 IO ) / 논리적 IO ) * 100

- 캐시 탐색 메커니즘 : Direct Path IO 를 제외한 모든 블록 IO 는 메모리 버퍼캐시를 경유한다. 버퍼캐시 탐색과정을 거치는 오퍼레이션은 다음과 같다.

1.. 인덱스 루트 블록을 읽을때
2.. 인덱스 루트블록에서 얻은 주소정보로 브랜치 블록을 읽을 때
3.. 인덱스 브랜치 블록에서 얻은 주소정보로 리프 블록을 읽을 때
4.. 인덱스 리프 블록에서 얻은 주소정보로 테이블 블록을 읽을 때
5.. 테이블 블록을 Full scan 할 때

해시함수 - 해시체인 - 버퍼헤더 => 버퍼 블록

해시 알고리즘으로 버퍼 헤더를 찾고, 거기서 얻은 포린터로 버퍼 블록을 액세스하며, 체인에 없으면 디스크로부터 읽어서 체인에 연결한다. 따라서, 해시 체인 내에서는 정렬이 보장되지 않는다.

버퍼캐시는 SGA 구성요소 이므로 버퍼캐시에 캐싱된 버퍼블록은 공유자원이다. 따라서, 여러 프로세스가 읽으려 하면 블록 정합성에 문제가 생길 수 있어 직렬화 메커니즘을 사용한다. 이러한 줄세우기 메커니즘이 래치 이다.

cf) 캐시버퍼 체인 래치 : 대량의 데이터를 읽을 때 모든 블록에 대해 해시 체인을 탐색하는데 이때 다른 프로세스에 의해 체인 구조가 변경되면 안된다. 따라서, 해시 체인 앞쪽에 자물쇠를 달아 키를 획득한 프로세스만이 체인으로 진입하도록 되어있다.

또한, 같은 블록에 접근할 수 있으므로 래치를 해제하기 전에 버퍼 헤더에 버퍼 Lock 을 설정하여 직렬화 문제를 해결한다.

#************************* 인덱스 튜닝 ************************* #

1. 인덱스 튜닝에는 칼럼 정렬 순서도 중요하다. 따라서, 범위 조건을 선두에 놓지 않도록 조심한다.

2. 랜덤 액세스를 최소화 해야 테이블 액세스 횟수가 줄어든다. 인덱스 외 칼럼 정보 등으로 인해 랜덤 IO 가 생기지 않도록 해야한다.

ROWID = DBA + 로우 번호
DBA = 데이터 파일 번호 + 블록 번호
블록 번호 = 데이터 파일 내에서 부여한 상대적 순번
로우 번호 = 블록 내 순번

- 수직적 탐색은 조건을 만족하는 첫 번째 레코드를 찾는 과정이므로 인덱스 브랜치 블록에서 찾아도 해당 레코드 하위 블록으로 이동하면 안되고 바로 직전 레코드 (여기서는 LMC : LeftMost Child) 가 가리키는 하위 블록으로 이동 해야한다. ( 같거나 크면 이전 블록으로 내려간다 )

- 수평적 탐색 : 인덱스 리프 블록 끼리는 서로 앞뒤 블록에 대한 주소갑슬 갖는 양방향 연결리스트 구조로 되어 있다. 조건절을 만족하는 데이터를 모두 찾아 ROWID 를 얻어서 (인덱스 스캔) 테이블도 액세스 한다.

Balanced : Delete 등 작업으로 인해서 루트로 부터 리프 블록까지의 높이가 달라질 일은 없다, 즉 항상 같다.

- 인덱스 선두 컬럼을 가공하면 스캔 시작점을 찾을 수 없고 멈출 수도 없어 리프 블록 전체를 스캔해야만 하는 Index Full Scan 방식으로 작동한다.

- OR Expansion : OR 조건식을 사용하는 경우, 인덱스 Range Scan 을 사용하기 위해 옵티마이저가 조건을 분리하여 1) 조건 SQL union all 2) 조건 SQL ( 1조건이 아니거나 1조건이 null 인 경우 ) 로 만들어 낼 수 있는데 이를 OR Expansion 이라고 하며, use_concat 힌트를 이용했을 때, 이러한 현상이 주로 일어나고 실행계획에는 CONCATENATION 이라는 명칭으로 나타나게 된다.

cf) IN 조건도 위와 같이 union all 을 이용하여 Index Range Scan 이 사용되기도 한다.

- 인덱스 선두칼럼의 순서에 따라 , 인덱스 스캔 효율이 매우 떨어지기도 한다. ( Range Scan 이 항상 좋은 건 X )

- 인덱스 리프 블록은 양방향 연결 리스트 구조이므로 내림차순 정렬에도 인덱스를 활용할 수 있다.

- ORDER BY , SELECT LIST 에서 칼럼 가공 시 ?

1) 예를 들어, 조건절 - ORDER BY 절 까지 칼럼 순서 = 인덱스 칼럼 순서 일 경우에 ORDER BY A||B 과 같은 식으로 가공 값 정렬을 요청한다면 정렬 연산을 생략할 수 없다.

2) 선두칼럼 = 조건과 이후 후행 칼럼 범위 조건 , ORDER BY 절 사용 등으로 인덱스 칼럼 순서와 동일하게 SQL 이 작성되어도, SELECT 절에 TO_CHAR 와 같은 함수로 가공을 할 경우 ( 그리고 테이블 ALIAS 는 생략한 채 그 칼럼 ALIAS 으로 정렬을 할 경우 ) 정렬 연산을 생략할 수 없다.


- FIRST ROW : 인덱스를 잘 이용하는 경우, MIN/MAX 값을 인덱스 리프 블록 중 레코드 하나만 읽고 멈출 수 있는데 ( 실행계획에 FIRST ROW ) , TO_NUMBER 와 같은 함수를 쓰게 될 경우, 정렬 연산은 생략할 수 없게 된다.

* MAX 변경일자의 MAX 변경순번을 구하려고 하는 경우, 해당 레코드 단위로 검사해야하기 때문에 테이블을 여러번 스캔해야할 수도 있다, 이러한 경우에는 || 를 이용해서 한번에 구한 후에 SUBSTR 할 수 있다. 하지만 이 경우에도 칼럼 가공으로 인한 오버로드는 생길 수 있다. 이 경우에는 TOP - N 알고리즘을 사용해야 한다.
