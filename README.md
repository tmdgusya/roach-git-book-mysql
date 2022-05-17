# Full-Text Search

MySQL 에는 기존에는 MyISAM 에서만 이용 가능한 Full-Text Index 가 존재하였다. MySQL 5.6 부터 InnoDB 에도 Full-Text Search 기능이 지원되기 시작했다.&#x20;

### MySQL 의 토큰 분리 알고리즘

* 형태소 분석
* n-gram 파서

별도로 MeCab 을 플러그인으로 적용가능하나 이와 관련된 내용은 해당 위키에서는 다루지 않을 것이다.

### 형태소 분석

형태소 분석이란 문장을 공백과 뛰어쓰기로 분리하고, 각 단어의 조사를 제거해서 명사 도는 어근을 찾아서 인덱싱 하는 방법이다. 하지만 MySQL 서버에는 이와 같은 기능을 구현하고 있지는 않다.

### n-gram

n-gram 은 문장 자체를 공백과 뛰어쓰기로 단어를 구분하고, ngram\_token\_size 에 맞게 쪼개서 단순하게 인덱싱 하는 방법이다. 테이블에 ngram index 를 적용하기 위해서는\*\* with parser ngram\*\* 을 반드시 함께 붙여주어야 한다.

```sql
CREATE TABLE articles ( 
    id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY, 
    title VARCHAR(200), body TEXT, 
    FULLTEXT (title,body) WITH PARSER ngram 
) ENGINE=InnoDB CHARACTER SET utf8mb4;
```

### ngram\_token\_size

ngram\_token\_size 란 시스템 변수로 **기본값은 2 이며, 1 \~ 10 사이의 숫자로 설정 가능**하다. \ ngram\_token\_size 에 따라서 검색에 영향을 미친다. 예를 들면 ngram\_token\_size 를 2로 설정했을 경우 1가지 단어로 검색을 질의하는 것은 의미가 없는 결과를 도출한다. 따라서 잘 설정해야 한다.

```sql
# ngram_toke_size = 2 SELECT COUNT(*) FROM articles 
# WHERE MATCH (title,body) AGAINST ('db' IN BOOLEAN MODE); // result = 1
SELECT COUNT(*) FROM articles 
WHERE MATCH (title,body) AGAINST ('d' IN BOOLEAN MODE); // result = 0
```

### StopWord

FullText Search 기능을 이용할때 가장 중요한 기능인데 StopWord 의 개념을 알아야 한다. \ 금칙어의 개념으로 **해당 Stopword 와 관련된 부분은 질의 문에서 무시되거나, 인덱싱이 안될수 있다.**&#x20;

\[공식문서]\(https://dev.mysql.com/doc/refman/8.0/en/fulltext-stopwords.html)

stop\_word list 는 Inno DB 의 경우 아래 쿼리로 확인 가능하다.

```sql
SELECT * FROM INFORMATION_SCHEMA.INNODB_FT_DEFAULT_STOPWORD;
```

예를 들면 아래와 같은 현상이다.

#### stopword 옵션 종료 전 쿼리 및 결과

```sql
SELECT COUNT(*) FROM person WHERE MATCH (name) AGAINST ('ach' IN BOOLEAN MODE); // result = 2
SELECT COUNT(*) FROM person WHERE MATCH (name) AGAINST ('ch' IN BOOLEAN MODE); // result = 2
```

#### stopword 옵션 종료 후 쿼리 및 결과

```sql
SELECT COUNT(*) FROM person WHERE MATCH (name) AGAINST ('ach' IN BOOLEAN MODE); // result = 1
SELECT COUNT(*) FROM person WHERE MATCH (name) AGAINST ('ch' IN BOOLEAN MODE); // result = 2
```

보통 정관사와 같은 관용표현들이 많이 등록되어 있어 'a' 가 질의문에서 무시된 것을 알 수 있다. \ 즉 'ach' 로 질의하나 'ch' 로 질의하나 똑같은 결과를 받을 수 밖에 없었던 것이다. \\

### FullText Search Query Mode

MySQL 서버의 전문 검색 쿼리모드는 두가지가 존재한다.

* NATURAL LANGUAGE MODE
* BOOLEAN MODE

#### NATURAL LANGUAGE MODE

MySQL 의 NATURAL LANGUAGE MODE 는 검색어에서 키워드를 추출한뒤 검색어에 포함하는 방법이다. 기본적으로 어떠한 모드도 입력하지 않는다면 해당 모드로 쿼리가 질의된다. 자연어 검색은 인간이 사용하는 언어처럼 해석되어 질의된다. 따라서 영어 조사 성격의 StopWord 들이 적용되므로 조심해야 한다.

특별한 연산자는 존재하지 않고 예외적으로 하나의 단어로 보게 하는 **"" (double quote)** 를 지원한다.

예를 들면, 아래와 같이 데이터를 세팅하고 검색한다고 해보자.

`mysql> INSERT INTO articles (title,body) VALUES ('MySQL Tutorial','DBMS stands for DataBase ...'), ('How To Use MySQL Well','After you went through a ...'), ('Optimizing MySQL','In this tutorial, we show ...'), ('1001 MySQL Tricks','1. Never run mysqld as root. 2. ...'), ('MySQL vs. YourSQL','In the following database comparison ...'), ('MySQL Security','When configured properly, MySQL ...');`

`+----+-------------------+------------------------------------------+ | id | title | body | +----+-------------------+------------------------------------------+ | 1 | MySQL Tutorial | DBMS stands for DataBase ... | | 5 | MySQL vs. YourSQL | In the following database comparison ... | +----+-------------------+------------------------------------------+`\


이렇게 단어 말고도 문장 자체를 검색하는데 이용 가능하다. 이런 경우를 MySQL Menual 에서는 **"Phrase Search"** 라고 명명한다.&#x20;

`mysql> SELECT id, body, MATCH (title,body) AGAINST ('Security implications of running MySQL as root' IN NATURAL LANGUAGE MODE) AS score FROM articles WHERE MATCH (title,body) AGAINST ('Security implications of running MySQL as root' IN NATURAL LANGUAGE MODE);`

#### IN BOOLEAN MODE

BOOLEAN MODE 는 질의문에 사용되는 키워드에 연산조건의 활용이 가능하다.&#x20;

`mysql> SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+MySQL -YourSQL' IN BOOLEAN MODE);`

위의 쿼리는 MySQL 은 포함하지만 YourSQL 은 포함하지 않는 경우이다. \ 연산에는 '+', '-' 외에도 패턴 검색이 가능하다.&#x20;

```sql
mysql> SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('My*' IN BOOLEAN MODE); // wildcard
```

이 외에도 지원하는 연산자가 많다. 만약 더 확인해보고 싶다면 아래 링크 에서 확인하면 된다.

* https://dev.mysql.com/doc/refman/8.0/en/fulltext-boolean.html

\===== 실제 적용기 =====

MySQL 을 통한 이메일 자동검색

* [https://devroach.tistory.com/87](https://devroach.tistory.com/87)
