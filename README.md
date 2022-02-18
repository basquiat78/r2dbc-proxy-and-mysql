# Webflux Query Logging

가장 기본적인 방법은 application.yml에 다음과 같이 설정하는 것이다.

```
logging:
  level:
    root: info
    org:
      springframework:
        r2dbc: DEBUG
```

하지만 이렇게 되면 그냥 쿼리가 어떻게 날아가는지에 대해서만 알 수 있으며 바인딩된 변수와 쿼리 결과를 알수 가 없다.

그래서 r2dbc-proxy를 사용한다.

그레이들 기준으로 다음과 같이 추가를 하자.

```
implementation 'io.r2dbc:r2dbc-proxy:0.9..RELEASE'
```

그리고 나서 

```
package io.basquiat.boards.common.listener;

import io.r2dbc.proxy.core.QueryExecutionInfo;
import io.r2dbc.proxy.listener.ProxyExecutionListener;
import io.r2dbc.proxy.support.QueryExecutionInfoFormatter;
import lombok.extern.slf4j.Slf4j;

/**
 * r2dbc proxy를 활용한 쿼리 로그
 */
@Slf4j
public class QueryLoggingListener implements ProxyExecutionListener {

    @Override
    public void afterQuery(QueryExecutionInfo execInfo) {
        QueryExecutionInfoFormatter formatter = new QueryExecutionInfoFormatter().addConsumer((info, sb) -> {
                                                                                        sb.append("ConnectionId: ");
                                                                                        sb.append(info.getConnectionInfo().getConnectionId());
                                                                                 })
                                                                                 .newLine()
                                                                                 .showQuery()
                                                                                 .newLine()
                                                                                 .showBindings()
                                                                                 .newLine()
                                                                                 .addConsumer((info, sb) -> {
                                                                                        sb.append("Result Count : ");
                                                                                        sb.append(info.getCurrentResultCount());
                                                                                 });
        log.info(formatter.format(execInfo));
    }

    @Override
    public void eachQueryResult(QueryExecutionInfo execInfo) {
        QueryExecutionInfoFormatter formatter = new QueryExecutionInfoFormatter().addConsumer((info, sb) -> {
                                                                                        sb.append("Result Row : ");
                                                                                        sb.append(info.getCurrentMappedResult());
                                                                                 });
        log.info(formatter.format(execInfo));
    }

}
```
다음과 같이 ProxyExecutionListener를 상속받아 구현을 하면 된다.

쿼리 결과는 따로 eachQueryResult를 오버라이딩해서 구현하면 된다.

그리고 관련 리스너를 만들었다면 application.yml에 다음과 같이 url부분을 수정해 줘야 한다.


```
mysql:
  host: localhost
  port: 3306
  username: root
  password:
  database: basquiat
  proxy-listener: io.basquiat.boards.common.listener.QueryLoggingListener

spring:
  config:
    activate:
      on-profile: local
  r2dbc:
    url: r2dbc:proxy:mysql://${mysql.host}:${mysql.port}/${mysql.database}?proxyListener=${mysql.proxy-listener}&useSSL=false&useUnicode=yes&characterEncoding=UTF-8&serverTimezone=Asia/Seoul
    username: ${mysql.username}
    password: ${mysql.password}
    pool:
      enabled: true
      initial-size: 10
      max-size: 30
      max-idle-time: 30m
      validation-query: SELECT 1

```
url 부분을 보면 query param에 proxyListener를 추가해주고 해당 리스너의 풀패키명을 포함한 클래스명을 작성해서 보내주면 된다.     

또한 :proxy: 이 부분을 추가해 주면 완료된다.

좀 더 자세한 방법은 다음 공식 github에서 찾아보자.     

공식 홈페이지에서는 PostgreSQL가 예제로 사용된다.    

[r2dbc-proxy](https://github.com/r2dbc/r2dbc-proxy)

# fetch()에 대한 버그 리포트    

PostgreSQL이나 h2의 경우에는 이런 경우가 발생하지 않는데 mySql을 사용하기 위해 dev.miku:r2dbc-mysql 라이브러리를 사용하게 되면 fetch()가 이상하게 작동하는 버그가 있다.     

fetch()는 실제로 map을 통해 Row객체와 RowMetadata객체를 이용해 Map<String, Object>로 변환하는 메소드이다.     

실제로 구현된 부분을 살펴보면 

```
/*
 * Copyright 2002-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.r2dbc.core;

import java.util.Collection;
import java.util.Map;
import java.util.function.BiFunction;

import io.r2dbc.spi.ColumnMetadata;
import io.r2dbc.spi.Row;
import io.r2dbc.spi.RowMetadata;

import org.springframework.lang.Nullable;
import org.springframework.util.LinkedCaseInsensitiveMap;

/**
 * {@link BiFunction Mapping function} implementation that creates a
 * {@code java.util.Map} for each row, representing all columns as
 * key-value pairs: one entry for each column, with the column name as key.
 *
 * <p>The Map implementation to use and the key to use for each column
 * in the column Map can be customized through overriding
 * {@link #createColumnMap} and {@link #getColumnKey}, respectively.
 *
 * <p><b>Note:</b> By default, ColumnMapRowMapper will try to build a linked Map
 * with case-insensitive keys, to preserve column order as well as allow any
 * casing to be used for column names. This requires Commons Collections on the
 * classpath (which will be autodetected). Else, the fallback is a standard linked
 * HashMap, which will still preserve column order but requires the application
 * to specify the column names in the same casing as exposed by the driver.
 *
 * @author Mark Paluch
 * @since 5.3
 */
public class ColumnMapRowMapper implements BiFunction<Row, RowMetadata, Map<String, Object>> {

	/** A default {@code ColumnMapRowMapper} instance. */
	public final static ColumnMapRowMapper INSTANCE = new ColumnMapRowMapper();


	@Override
	public Map<String, Object> apply(Row row, RowMetadata rowMetadata) {
		Collection<String> columns = rowMetadata.getColumnNames();
		int columnCount = columns.size();
		Map<String, Object> mapOfColValues = createColumnMap(columnCount);
		int index = 0;
		for (String column : columns) {
			String key = getColumnKey(column);
			Object obj = getColumnValue(row, index++);
			mapOfColValues.put(key, obj);
		}
		return mapOfColValues;
	}

	/**
	 * Create a {@link Map} instance to be used as column map.
	 * <p>By default, a linked case-insensitive Map will be created.
	 * @param columnCount the column count, to be used as initial capacity for the Map
	 * @return the new {@link Map} instance
	 * @see LinkedCaseInsensitiveMap
	 */
	protected Map<String, Object> createColumnMap(int columnCount) {
		return new LinkedCaseInsensitiveMap<>(columnCount);
	}

	/**
	 * Determine the key to use for the given column in the column {@link Map}.
	 * @param columnName the column name as returned by the {@link Row}
	 * @return the column key to use
	 * @see ColumnMetadata#getName()
	 */
	protected String getColumnKey(String columnName) {
		return columnName;
	}

	/**
	 * Retrieve a R2DBC object value for the specified column.
	 * <p>The default implementation uses the {@link Row#get(int)} method.
	 * @param row is the {@link Row} holding the data
	 * @param index is the column index
	 * @return the Object returned
	 */
	@Nullable
	protected Object getColumnValue(Row row, int index) {
		return row.get(index);
	}

}
```
이것을 사용하게 된다. 이 코드의 방식은 만들어진 쿼리의 컬럼을 list 객체에 담고 Row객체에서 인덱스를 통해서 가져오게 된다.     

도식으로 표현한면

```
id | name | age |
 1 | Foo  | 30 
 2 | Bar  | 31

```
처럼 결과값이 나왔다면 

컬럼을 {"id", "name", "age"}로 리스트에 담고 index를 통해서 Row객체로부터 값을 가져와 매핑을 하면

```
id=1, name=Foo, age=30
id=2, name=Bar, age=31
```
처럼 LinkedCaseInsensitiveMap객체로 반환하게 된다.     

하지만 dev.miku:r2dbc-mysql에서는 RowMetadata을 구현한 MySqlRowMetadata 에서 이것을 좀 이상하게 처리한다.    

```
/*
 * Copyright 2018-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package dev.miku.r2dbc.mysql;

import dev.miku.r2dbc.mysql.message.server.DefinitionMetadataMessage;
import dev.miku.r2dbc.mysql.util.InternalArrays;
import io.r2dbc.spi.RowMetadata;

import java.util.Arrays;
import java.util.Comparator;
import java.util.List;
import java.util.NoSuchElementException;
import java.util.Set;

import static dev.miku.r2dbc.mysql.util.AssertUtils.requireNonNull;

/**
 * An implementation of {@link RowMetadata} for MySQL database text/binary results.
 *
 * @see MySqlNames column name searching rules.
 */
final class MySqlRowMetadata implements RowMetadata {

    private static final Comparator<MySqlColumnMetadata> NAME_COMPARATOR = (left, right) ->
        MySqlNames.compare(left.getName(), right.getName());

    private final MySqlColumnMetadata[] idSorted;

    private final MySqlColumnMetadata[] nameSorted;

    /**
     * Copied column names from {@link #nameSorted}.
     */
    private final String[] names;

    private final ColumnNameSet nameSet;

    private MySqlRowMetadata(MySqlColumnMetadata[] idSorted) {
        int size = idSorted.length;

        if (size <= 0) {
            throw new IllegalArgumentException("least 1 column metadata");
        }

        MySqlColumnMetadata[] nameSorted = new MySqlColumnMetadata[size];
        System.arraycopy(idSorted, 0, nameSorted, 0, size);
        Arrays.sort(nameSorted, NAME_COMPARATOR);

        this.idSorted = idSorted;
        this.nameSorted = nameSorted;
        this.names = getNames(nameSorted);
        this.nameSet = new ColumnNameSet(this.names);
    }

    @Override
    public MySqlColumnMetadata getColumnMetadata(int index) {
        if (index < 0 || index >= idSorted.length) {
            throw new ArrayIndexOutOfBoundsException(String.format("column index %d is invalid, total %d", index, idSorted.length));
        }

        return idSorted[index];
    }

    @Override
    public MySqlColumnMetadata getColumnMetadata(String name) {
        requireNonNull(name, "name must not be null");

        int index = MySqlNames.nameSearch(this.names, name);

        if (index < 0) {
            throw new NoSuchElementException(String.format("column name '%s' does not exist in %s", name, Arrays.toString(this.names)));
        }

        return nameSorted[index];
    }

    @Override
    public List<MySqlColumnMetadata> getColumnMetadatas() {
        return InternalArrays.asImmutableList(idSorted);
    }

    @Override
    public Set<String> getColumnNames() {
        return nameSet;
    }

    @Override
    public String toString() {
        return String.format("MySqlRowMetadata{metadata=%s, sortedNames=%s}", Arrays.toString(idSorted), nameSet);
    }

    MySqlColumnMetadata[] unwrap() {
        return idSorted;
    }

    static MySqlRowMetadata create(DefinitionMetadataMessage[] columns) {
        int size = columns.length;
        MySqlColumnMetadata[] metadata = new MySqlColumnMetadata[size];

        for (int i = 0; i < size; ++i) {
            metadata[i] = MySqlColumnMetadata.create(i, columns[i]);
        }

        return new MySqlRowMetadata(metadata);
    }

    private static String[] getNames(MySqlColumnMetadata[] metadata) {
        int size = metadata.length;
        String[] names = new String[size];

        for (int i = 0; i < size; ++i) {
            names[i] = metadata[i].getName();
        }

        return names;
    }
}

```
코드를 보면 알겠지만 컬럼 네임을 소팅을 한다.!!!!!!      

~~아니 왜!!!~~

```
id | name | age |
 1 | Foo  | 30 
 2 | Bar  | 31
```

이 부분을 다시 보자.

저렇게 되면 다음과 같이 작동을 하게 된다.      

컬럼을 {"age", "id", "name"}으로 리스트를 담고 index를 통해서 Row객체로부터 값을 가져와 매핑을 하면

```
age=1, id=Foo, name=30
age=2, id=Bar, name=31
```

우연찮게 쿼리를 다음과 같이

```
SELECT age, 
       id, 
       name
  FROM USER
```
처럼 작성을 했다면야 문제가 안되겠지만 보통은

```
SELECT id,
       name,
       age
  FROM USER
```
이렇게 작성하지 않나?     

그래서 실제로 

```
SELECT id AS 1_id,
       name AS 2_name,
       age AS 3_age
  FROM USER
```
처럼 말도 안되게 별칭을 줘야 한다.      

~~컬럼이 많으면 저렇게 별칭을 주는 것은 미친 짓이다....~~

우연찮게 쿼리 작성을 하고 보니 소팅이 맞아떨어져 제대로 나올 수도 있지만....     

당연히 저렇게 되면 아스테릭 문자인 '*'을 사용하는 것은 어불성설이고 테이블 조인시 Projection을 믿을 수가 없게 된다.....

따라서 이것을 해결하는 방법은 직접적으로 Row와 RowMatadata를 통해 Map객체로 반환하는 mapper를 만들던가 아니면 이넘을 fork를 떠서 직접 수정하고 PR를 날리는 수밖에 없다.     

일단 그래서 급하게 사용해야 한다면 다음과 같이

```
package io.basquiat.boards.common.mapper;

import io.r2dbc.spi.Row;
import io.r2dbc.spi.RowMetadata;
import org.springframework.stereotype.Component;
import org.springframework.util.LinkedCaseInsensitiveMap;

import java.util.Collection;
import java.util.Map;
import java.util.function.BiFunction;

/**
 * RowToMapMapper
 *
 * MySqlRowMetadata에서 컬럼을 이름으로 소팅하는 부분이 있어서 그냥 날코딩으로 fetch대신 이것을 사용한다.
 *
 * created by basquiat
 *
 */
@Component
public class RowToMapMapper implements BiFunction<Row, RowMetadata, Map<String, Object>> {

    /**
     * 순수한 형태로 row to Map으로 변환한다.
     * @param row
     * @param rowMetadata
     * @return Map<String, Object>
     */
    @Override
    public Map<String, Object> apply(Row row, RowMetadata rowMetadata) {
        Collection<String> columns = rowMetadata.getColumnNames();
        int columnCount = columns.size();
        Map<String, Object> mapOfColValues = new LinkedCaseInsensitiveMap<>(columnCount);
        for (String column : columns) {
            String key = column;
            Object obj = row.get(key);
            mapOfColValues.put(key, obj);
        }
        return mapOfColValues;
    }

}
```
을 만들고     


```
package io.basquiat.boards.music.repository.custom.impl;

import io.basquiat.boards.common.mapper.RowToMapMapper;
import io.basquiat.boards.music.domain.entity.Album;
import io.basquiat.boards.music.domain.entity.Label;
import io.basquiat.boards.music.domain.entity.Musician;
import io.basquiat.boards.music.repository.custom.CustomMusicianRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.r2dbc.core.R2dbcEntityTemplate;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import static io.basquiat.boards.common.utils.DateUtils.toDateTime;
import static io.basquiat.boards.common.utils.NumberUtils.parseLong;
import static java.util.stream.Collectors.toList;

/**
 * CustomMusicianRepository 구현체
 * created by basquiat
 */
@Slf4j
@RequiredArgsConstructor
public class CustomMusicianRepositoryImpl implements CustomMusicianRepository {

    private final R2dbcEntityTemplate query;

    private final RowToMapMapper rowToMapMapper;
 
    /**
     * 앨범 정보를 포함한 뮤지션들의 정보를 반환한다.
     * @return Flux<Musician>
     */
    @Override
    public Flux<Musician> findMusicians() {
        StringBuffer sb = new StringBuffer();
        sb.append("SELECT musician.id, ");
        sb.append("       musician.name, ");
        sb.append("       musician.instrument, ");
        sb.append("       musician.birth, ");
        sb.append("       musician.created_at AS created, ");
        sb.append("       musician.updated_at AS updated, ");
        sb.append("       album.id AS albumId, ");
        sb.append("       album.title AS title, ");
        sb.append("       album.release_year AS releaseYear, ");
        sb.append("       album.genre AS genre, ");
        sb.append("       label.id AS labelId, ");
        sb.append("       label.name AS labelName ");
        sb.append("     FROM musician");
        sb.append("     LEFT JOIN album ON musician.id = album.musician_id");
        sb.append("     JOIN label ON album.label_id = label.id");
        sb.append("     ORDER BY musician.id");

        return query.getDatabaseClient().sql(sb.toString())
                                        .map(rowToMapMapper::apply)
                                        .all()
                                        .bufferUntilChanged(result -> result.get("id"))
                                        .map(rows ->
                                                Musician.builder()
                                                        .id(parseLong(rows.get(0).get("id")))
                                                        .name(rows.get(0).get("name").toString())
                                                        .instrument(rows.get(0).get("instrument").toString())
                                                        .birth(rows.get(0).get("birth").toString())
                                                        .createdAt(toDateTime(rows.get(0).get("created")))
                                                        .updatedAt(toDateTime(rows.get(0).get("updated")))
                                                        .albums(rows.stream()
                                                                .map(row -> Album.builder()
                                                                                 .id(row.get("albumId").toString())
                                                                                 .title(row.get("title").toString())
                                                                                 .releaseYear(row.get("releaseYear").toString())
                                                                                 .genre(row.get("genre").toString())
                                                                                 .label(Label.builder()
                                                                                             .id(parseLong(row.get("labelId")))
                                                                                             .name(row.get("labelName").toString())
                                                                                 .build())
                                                                        .build())
                                                                .collect(toList()))
                                                        .build()
                                        );
    }

}

```
매퍼를 주입받아서 fetch()대신에 map()을 사용해서 처리해야 한다.       

급한대로 처리하고 차후 이것을 fork떠서 제안을 해 볼 생각이다....


# 그냥...

라이브러리 교체가 거시기하다면 위 방법대로 우선 사용하는 게 최우선이것 같다.     

하지만 교체가 가능하다면?

라이브러리 전격 교체      

```
implementation 'dev.miku:r2dbc-mysql:0.8.2.RELEASE'
```

스프링 커뮤니티에서 이 넘은 관리 안된지 오래되었다고 한다.      

```
implementation 'com.github.jasync-sql:jasync-r2dbc-mysql:2.0.6'
```

![속이 편안](https://github.com/basquiat78/r2dbc-proxy-and-mysql/blob/main/pyunan.jpeg)
