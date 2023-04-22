# 1.Mybatis源码中Builder模式

#### 一、Builder 模式： <a href="#fpkma" id="fpkma"></a>

Builder 模式：用于创建复杂对象。

**chatgpt:**

mybatis 是一个使用 java 编写的持久化框架，其中包含了建造者模式。建造者模式是一种创建复杂对象的设计模式，在 mybatis 中主要用于构建 sql 语句。mybatis 中建造者模式主要分为三个部分：

1. sqlsource.builder sqlsource 是存储 sql 语句的接口。在 mybatis 中，你可以通过构建 sqlsource 对象来生成可执行的 sql 语句。sqlsource.builder 就是一个建造者模式，它用于构建 sqlsource 对象。你可以通过调用 sqlsource.builder 中的方法来设置 sql 语句中的参数，例如 where 子句、set 子句等。
2. parametermapping.builder parametermapping 是用于描述 sql 语句参数信息的接口。在 mybatis 中，你可以通过构建 parametermapping 对象来生成 sql 参数的元数据。parametermapping.builder 也是一个建造者模式，它用于构建 parametermapping 对象。你可以通过调用 parametermapping.builder 中的方法来设置参数的名称、类型、jdbc 类型等信息。
3. mappedstatement.builder mappedstatement 是 mybatis 中定义的一个映射语句对象，它描述了一个 sql 语句以及参数映射信息。在 mybatis 中，你可以通过构建 mappedstatement 对象来实现 sql 的 crud 操作。mappedstatement.builder 也是一个建造者模式，它用于构建 mappedstatement 对象。你可以通过调用 mappedstatement.builder 中的方法来设置 sql 语句、参数映射信息、返回值类型等信息。

总之，在 mybatis 中建造者模式被广泛运用，通过构建 sqlsource、parametermapping 和 mappedstatement 对象来生成可执行的 sql 语句并进行数据库 crud 操作。

1.SqlSessionFactoryBuilder 是用于创建 SqlSessionFactory 实例的辅助类

```java
/**
* Builds {@link SqlSession} instances.
*
* @author Clinton Begin
*
* 建造者模式 构建出复杂的对象
*/
public class SqlSessionFactoryBuilder {

    public SqlSessionFactory build(Reader reader) {
        return build(reader, null, null);
    }

    public SqlSessionFactory build(Reader reader, String environment) {
        return build(reader, environment, null);
    }

    public SqlSessionFactory build(Reader reader, Properties properties) {
        return build(reader, null, properties);
    }

    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        try {
            //
            XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
            //解析xml 生成 SqlSessionFactory
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                if (reader != null) {
                    reader.close();
                }
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }

    public SqlSessionFactory build(InputStream inputStream) {
        return build(inputStream, null, null);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment) {
        return build(inputStream, environment, null);
    }

    public SqlSessionFactory build(InputStream inputStream, Properties properties) {
        return build(inputStream, null, properties);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        try {
            XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                if (inputStream != null) {
                    inputStream.close();
                }
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }

    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }

}
```

2.SqlSourceBuilder是MyBatis中一个重要的构建SQL语句的工具类、是MyBatis中用来构建SQL语句的类，它负责将Mapper XML文件中定义的SQL语句转化成对应的SqlSource对象。

```java
package org.apache.ibatis.builder;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.StringTjkenizer;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.mapping.SqlSource;
import org.apache.ibatis.parsing.GenericTokenParser;
import org.apache.ibatis.parsing.TokenHandler;
import org.apache.ibatis.reflection.MetaClass;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.type.JdbcType;

/**
* @author Clinton Begin
* parse: 根据配置的sql节点创建SqlSource对象。
* parseDynamicTags: 解析含有动态标签的SQL语句。
*/
public class SqlSourceBuilder extends BaseBuilder {

    private static final String PARAMETER_PROPERTIES = "javaType,jdbcType,mode,numericScale,resultMap,typeHandler,jdbcTypeName";

    /**
*
*
* @param configuration
*/
    public SqlSourceBuilder(Configuration configuration) {
        super(configuration);
    }

    public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {


        ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType,
                                                                                additionalParameters);

        //解析并替换动态SQL中的占位符（如#{}和${}）
        GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
        //解析动态sql
        String sql;
        if (configuration.isShrinkWhitespacesInSql()) {
            sql = parser.parse(removeExtraWhitespaces(originalSql));
        } else {
            sql = parser.parse(originalSql);
        }
        //将动态SQL转换为SqlSource对象。
        return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
    }

    public static String removeExtraWhitespaces(String original) {
        StringTokenizer tokenizer = new StringTokenizer(original);
        StringBuilder builder = new StringBuilder();
        boolean hasMoreTokens = tokenizer.hasMoreTokens();
        while (hasMoreTokens) {
            builder.append(tokenizer.nextToken());
            hasMoreTokens = tokenizer.hasMoreTokens();
            if (hasMoreTokens) {
                builder.append(' ');
            }
        }
        return builder.toString();
    }

    /**
* 处理参数映射的类。它主要用于将 #{paramName} 形式的参数占位符转换成对应的参数值，然后与 SQL 语句合并成完整的可执行 SQL 语句。
*/
    private static class ParameterMappingTokenHandler extends BaseBuilder implements TokenHandler {

        private final List<ParameterMapping> parameterMappings = new ArrayList<>();
        private final Class<?> parameterType;
        private final MetaObject metaParameters;

        public ParameterMappingTokenHandler(Configuration configuration, Class<?> parameterType,
        Map<String, Object> additionalParameters) {
        super(configuration);
        this.parameterType = parameterType;
        this.metaParameters = configuration.newMetaObject(additionalParameters);
        }

        public List<ParameterMapping> getParameterMappings() {
        return parameterMappings;
        }

        /**
        * 将参数占位符转换成对应的参数值。这里的 content 参数实际上就是 #{}  占位符中的 paramName
        * @param content
        * @return
        */
        @Override
        public String handleToken(String content) {
        parameterMappings.add(buildParameterMapping(content));
        return "?";
        }

        private ParameterMapping buildParameterMapping(String content) {
        Map<String, String> propertiesMap = parseParameterMapping(content);
        String property = propertiesMap.get("property");
        Class<?> propertyType;
        if (metaParameters.hasGetter(property)) { // issue #448 get type from additional params
        propertyType = metaParameters.getGetterType(property);
        } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
        } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
        } else if (property == null || Map.class.isAssignableFrom(parameterType)) {
        propertyType = Object.class;
        } else {
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        if (metaClass.hasGetter(property)) {
        propertyType = metaClass.getGetterType(property);
        } else {
        propertyType = Object.class;
        }
        }
        ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
        Class<?> javaType = propertyType;
        String typeHandlerAlias = null;
        for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
        javaType = resolveClass(value);
        builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
        builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {
        builder.mode(resolveParameterMode(value));
        } else if ("numericScale".equals(name)) {
        builder.numericScale(Integer.valueOf(value));
        } else if ("resultMap".equals(name)) {
        builder.resultMapId(value);
        } else if ("typeHandler".equals(name)) {
        typeHandlerAlias = value;
        } else if ("jdbcTypeName".equals(name)) {
        builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
        // Do Nothing
        } else if ("expression".equals(name)) {
        throw new BuilderException("Expression based parameters are not supported yet");
        } else {
        throw new BuilderException("An invalid property '" + name + "' was found in mapping #{" + content
        + "}.  Valid properties are " + PARAMETER_PROPERTIES);
        }
        }
        if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
        }
        return builder.build();
        }

        private Map<String, String> parseParameterMapping(String content) {
        try {
        return new ParameterExpression(content);
        } catch (BuilderException ex) {
        throw ex;
        } catch (Exception ex) {
        throw new BuilderException("Parsing error was found in mapping #{" + content
        + "}.  Check syntax #{property|(expression), var1=value1, var2=value2, ...} ", ex);
        }
        }
        }

        }
```

3.ParameterMapping是用于映射“#{}”和“${}”占位符的对象。

```java
/**
 * @author Clinton Begin
 */
public class ParameterMapping {

  private Configuration configuration;

  /**
   * 映射到参数的属性名称
   */
  private String property;
  /**
   * IN/OUT模式，决定如何传递参数
   */
  private ParameterMode mode;
  /**
   * 参数的类型
   */
  private Class<?> javaType = Object.class;

  /**
   * SQL类型。
   */
  private JdbcType jdbcType;

  /**
   *数字小数点的比例，仅在JDBC类型为DECIMAL或NUMERIC时有意义。
   */
  private Integer numericScale;


  private TypeHandler<?> typeHandler;

  /**
   * 结果集映射ID
   */
  private String resultMapId;


  private String jdbcTypeName;
  /**
   * OGNL表达式
   */
  private String expression;

  private ParameterMapping() {
  }

  public static class Builder {
    private final ParameterMapping parameterMapping = new ParameterMapping();

    public Builder(Configuration configuration, String property, TypeHandler<?> typeHandler) {
      parameterMapping.configuration = configuration;
      parameterMapping.property = property;
      parameterMapping.typeHandler = typeHandler;
      parameterMapping.mode = ParameterMode.IN;
    }

    public Builder(Configuration configuration, String property, Class<?> javaType) {
      parameterMapping.configuration = configuration;
      parameterMapping.property = property;
      parameterMapping.javaType = javaType;
      parameterMapping.mode = ParameterMode.IN;
    }

    public Builder mode(ParameterMode mode) {
      parameterMapping.mode = mode;
      return this;
    }

    public Builder javaType(Class<?> javaType) {
      parameterMapping.javaType = javaType;
      return this;
    }

    public Builder jdbcType(JdbcType jdbcType) {
      parameterMapping.jdbcType = jdbcType;
      return this;
    }

    public Builder numericScale(Integer numericScale) {
      parameterMapping.numericScale = numericScale;
      return this;
    }

    public Builder resultMapId(String resultMapId) {
      parameterMapping.resultMapId = resultMapId;
      return this;
    }

    public Builder typeHandler(TypeHandler<?> typeHandler) {
      parameterMapping.typeHandler = typeHandler;
      return this;
    }

    public Builder jdbcTypeName(String jdbcTypeName) {
      parameterMapping.jdbcTypeName = jdbcTypeName;
      return this;
    }

    public Builder expression(String expression) {
      parameterMapping.expression = expression;
      return this;
    }

    public ParameterMapping build() {
      resolveTypeHandler();
      validate();
      return parameterMapping;
    }

    private void validate() {
      if (ResultSet.class.equals(parameterMapping.javaType)) {
        if (parameterMapping.resultMapId == null) {
          throw new IllegalStateException("Missing resultmap in property '" + parameterMapping.property + "'.  "
              + "Parameters of type java.sql.ResultSet require a resultmap.");
        }
      } else if (parameterMapping.typeHandler == null) {
        throw new IllegalStateException("Type handler was null on parameter mapping for property '"
            + parameterMapping.property + "'. It was either not specified and/or could not be found for the javaType ("
            + parameterMapping.javaType.getName() + ") : jdbcType (" + parameterMapping.jdbcType + ") combination.");
      }
    }

    private void resolveTypeHandler() {
      if (parameterMapping.typeHandler == null && parameterMapping.javaType != null) {
        Configuration configuration = parameterMapping.configuration;
        TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
        parameterMapping.typeHandler = typeHandlerRegistry.getTypeHandler(parameterMapping.javaType,
            parameterMapping.jdbcType);
      }
    }

  }

  public String getProperty() {
    return property;
  }

  /**
   * Used for handling output of callable statements.
   *
   * @return the mode
   */
  public ParameterMode getMode() {
    return mode;
  }

  /**
   * Used for handling output of callable statements.
   *
   * @return the java type
   */
  public Class<?> getJavaType() {
    return javaType;
  }

  /**
   * Used in the UnknownTypeHandler in case there is no handler for the property type.
   *
   * @return the jdbc type
   */
  public JdbcType getJdbcType() {
    return jdbcType;
  }

  /**
   * Used for handling output of callable statements.
   *
   * @return the numeric scale
   */
  public Integer getNumericScale() {
    return numericScale;
  }

  /**
   * Used when setting parameters to the PreparedStatement.
   *
   * @return the type handler
   */
  public TypeHandler<?> getTypeHandler() {
    return typeHandler;
  }

  /**
   * Used for handling output of callable statements.
   *
   * @return the result map id
   */
  public String getResultMapId() {
    return resultMapId;
  }

  /**
   * Used for handling output of callable statements.
   *
   * @return the jdbc type name
   */
  public String getJdbcTypeName() {
    return jdbcTypeName;
  }

  /**
   * Expression 'Not used'.
   *
   * @return the expression
   */
  public String getExpression() {
    return expression;
  }

  @Override
  public String toString() {
    final StringBuilder sb = new StringBuilder("ParameterMapping{");
    // sb.append("configuration=").append(configuration); // configuration doesn't have a useful .toString()
    sb.append("property='").append(property).append('\'');
    sb.append(", mode=").append(mode);
    sb.append(", javaType=").append(javaType);
    sb.append(", jdbcType=").append(jdbcType);
    sb.append(", numericScale=").append(numericScale);
    // sb.append(", typeHandler=").append(typeHandler); // typeHandler also doesn't have a useful .toString()
    sb.append(", resultMapId='").append(resultMapId).append('\'');
    sb.append(", jdbcTypeName='").append(jdbcTypeName).append('\'');
    sb.append(", expression='").append(expression).append('\'');
    sb.append('}');
    return sb.toString();
  }
}
```

```xml
<select id="getUserByAge" parameterType="int" resultType="User">
    select * from user where age = #{age}
</select>
```

在上面的示例中，当Mybatis解析“#{age}”占位符时，它将创建一个ParameterMapping对象来映射参数。该ParameterMapping对象将具有以下属性值：

* property: "value"
* jdbcType: "INTEGER"
* javaType: "int"
* mode: "IN"

因此，当执行上述SQL语句时，Mybatis将使用ParameterMapping对象来填充SQL语句中的参数

4.MappedStatement用于描述 sql 映射文件中定义的一条 sql 语句。例如：id、参数类型、返回值类型、sql语句、缓存策略等等。

在mybatis框架中，mappedstatement builder主要有以下几个作用

1. 解析mapper.xml文件中的sql语句

mappedstatement builder通过解析mapper.xml文件中的sql节点，获取到该sql语句的相关信息，例如：id、返回值类型、参数类型、sql语句等等。在获取到这些信息之后，mappedstatement builder会将这些信息封装到mappedstatement对象中，并返回给框架使用。

2. 解析注解方式的sql语句

除了mapper.xml文件中的sql语句，mybatis还支持使用注解方式来编写sql语句。mappedstatement builder同样可以通过解析注解方式的sql语句，获取到该sql语句的相关信息，并将这些信息封装到mappedstatement对象中，返回给框架使用。

3. 生成cachekey对象

在mybatis框架中，为了提高查询性能，通常会使用缓存技术。mappedstatement builder可以根据mappedstatement对象中的信息，生成对应的cachekey对象，用于缓存查询结果。

4. 生成boundsql对象

在执行sql语句之前，mybatis框架需要将sql语句中的占位符替换为具体的参数值。mappedstatement builder可以根据传入的参数，生成对应的boundsql对象，该对象包含了替换完占位符后的sql语句和对应的参数值。

{% code lineNumbers="true" %}
```java
pjavan configuration;
  private String id;
  private Integer fetchSize;
  private Integer timeout;
  private StatementType statementType;
  private ResultSetType resultSetType;
  private SqlSource sqlSource;
  private Cache cache;
  private ParameterMap parameterMap;
  private List<ResultMap> resultMaps;
  private boolean flushCacheRequired;
  private boolean useCache;
  private boolean resultOrdered;
  private SqlCommandType sqlCommandType;
  private KeyGenerator keyGenerator;
  private String[] keyProperties;
  private String[] keyColumns;
  private boolean hasNestedResultMaps;
  private String databaseId;
  private Log statementLog;
  private LanguageDriver lang;
  private String[] resultSets;
  private boolean dirtySelect;

  MappedStatement() {
    // constructor disabled
  }

  public static class Builder {
    private final MappedStatement mappedStatement = new MappedStatement();

    public Builder(Configuration configuration, String id, SqlSource sqlSource, SqlCommandType sqlCommandType) {
      mappedStatement.configuration = configuration;
      mappedStatement.id = id;
      mappedStatement.sqlSource = sqlSource;
      mappedStatement.statementType = StatementType.PREPARED;
      mappedStatement.resultSetType = ResultSetType.DEFAULT;
      mappedStatement.parameterMap = new ParameterMap.Builder(configuration, "defaultParameterMap", null,
          new ArrayList<>()).build();
      mappedStatement.resultMaps = new ArrayList<>();
      mappedStatement.sqlCommandType = sqlCommandType;
      mappedStatement.keyGenerator = configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
      String logId = id;
      if (configuration.getLogPrefix() != null) {
        logId = configuration.getLogPrefix() + id;
      }
      mappedStatement.statementLog = LogFactory.getLog(logId);
      mappedStatement.lang = configuration.getDefaultScriptingLanguageInstance();
    }

    public Builder resource(String resource) {
      mappedStatement.resource = resource;
      return this;
    }

    public String id() {
      return mappedStatement.id;
    }

    public Builder parameterMap(ParameterMap parameterMap) {
      mappedStatement.parameterMap = parameterMap;
      return this;
    }

    public Builder resultMaps(List<ResultMap> resultMaps) {
      mappedStatement.resultMaps = resultMaps;
      for (ResultMap resultMap : resultMaps) {
        mappedStatement.hasNestedResultMaps = mappedStatement.hasNestedResultMaps || resultMap.hasNestedResultMaps();
      }
      return this;
    }

    public Builder fetchSize(Integer fetchSize) {
      mappedStatement.fetchSize = fetchSize;
      return this;
    }

    public Builder timeout(Integer timeout) {
      mappedStatement.timeout = timeout;
      return this;
    }

    public Builder statementType(StatementType statementType) {
      mappedStatement.statementType = statementType;
      return this;
    }

    public Builder resultSetType(ResultSetType resultSetType) {
      mappedStatement.resultSetType = resultSetType == null ? ResultSetType.DEFAULT : resultSetType;
      return this;
    }

    public Builder cache(Cache cache) {
      mappedStatement.cache = cache;
      return this;
    }

    public Builder flushCacheRequired(boolean flushCacheRequired) {
      mappedStatement.flushCacheRequired = flushCacheRequired;
      return this;
    }

    public Builder useCache(boolean useCache) {
      mappedStatement.useCache = useCache;
      return this;
    }

    public Builder resultOrdered(boolean resultOrdered) {
      mappedStatement.resultOrdered = resultOrdered;
      return this;
    }

    public Builder keyGenerator(KeyGenerator keyGenerator) {
      mappedStatement.keyGenerator = keyGenerator;
      return this;
    }

    public Builder keyProperty(String keyProperty) {
      mappedStatement.keyProperties = delimitedStringToArray(keyProperty);
      return this;
    }

    public Builder keyColumn(String keyColumn) {
      mappedStatement.keyColumns = delimitedStringToArray(keyColumn);
      return this;
    }

    public Builder databaseId(String databaseId) {
      mappedStatement.databaseId = databaseId;
      return this;
    }

    public Builder lang(LanguageDriver driver) {
      mappedStatement.lang = driver;
      return this;
    }

    public Builder resultSets(String resultSet) {
      mappedStatement.resultSets = delimitedStringToArray(resultSet);
      return this;
    }

    public Builder dirtySelect(boolean dirtySelect) {
      mappedStatement.dirtySelect = dirtySelect;
      return this;
    }

    /**
     * Resul sets.
     *
     * @param resultSet
     *          the result set
     *
     * @return the builder
     *
     * @deprecated Use {@link #resultSets}
     */
    @Deprecated
    public Builder resulSets(String resultSet) {
      mappedStatement.resultSets = delimitedStringToArray(resultSet);
      return this;
    }

    public MappedStatement build() {
      assert mappedStatement.configuration != null;
      assert mappedStatement.id != null;
      assert mappedStatement.sqlSource != null;
      assert mappedStatement.lang != null;
      mappedStatement.resultMaps = Collections.unmodifiableList(mappedStatement.resultMaps);
      return mappedStatement;
    }
  }

  public KeyGenerator getKeyGenerator() {
    return keyGenerator;
  }

  public SqlCommandType getSqlCommandType() {
    return sqlCommandType;
  }

  public String getResource() {
    return resource;
  }

  public Configuration getConfiguration() {
    return configuration;
  }

  public String getId() {
    return id;
  }

  public boolean hasNestedResultMaps() {
    return hasNestedResultMaps;
  }

  public Integer getFetchSize() {
    return fetchSize;
  }

  public Integer getTimeout() {
    return timeout;
  }

  public StatementType getStatementType() {
    return statementType;
  }

  public ResultSetType getResultSetType() {
    return resultSetType;
  }

  public SqlSource getSqlSource() {
    return sqlSource;
  }

  public ParameterMap getParameterMap() {
    return parameterMap;
  }

  public List<ResultMap> getResultMaps() {
    return resultMaps;
  }

  public Cache getCache() {
    return cache;
  }

  public boolean isFlushCacheRequired() {
    return flushCacheRequired;
  }

  public boolean isUseCache() {
    return useCache;
  }

  public boolean isResultOrdered() {
    return resultOrdered;
  }

  public String getDatabaseId() {
    return databaseId;
  }

  public String[] getKeyProperties() {
    return keyProperties;
  }

  public String[] getKeyColumns() {
    return keyColumns;
  }

  public Log getStatementLog() {
    return statementLog;
  }

  public LanguageDriver getLang() {
    return lang;
  }

  public String[] getResultSets() {
    return resultSets;
  }

  public boolean isDirtySelect() {
    return dirtySelect;
  }

  /**
   * Gets the resul sets.
   *
   * @return the resul sets
   *
   * @deprecated Use {@link #getResultSets()}
   */
  @Deprecated
  public String[] getResulSets() {
    return resultSets;
  }

  public BoundSql getBoundSql(Object parameterObject) {
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }

    return boundSql;
  }

  private static String[] delimitedStringToArray(String in) {
    if (in == null || in.trim().length() == 0) {
      return null;
    }
    return in.split(",");
  }

}
```
{% endcode %}
