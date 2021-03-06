## jDbPro
License: [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)   
  
jDbPro is a JDBC tool based on "Apache Commons DbUtils"(Hereinafter referred to as "DbUtils") but made below improvements:    
1.Add in-line style SQL support, "in-line" style using ThreadLocal to weave SQL parameters into a SQL String, easy to maintenance.  
2.Add SQL Template support, jDbPro allowed customize to use any Template and also have a simple template implementation.    
3.Improved DbUtils's Exception strategy, wrap SQLException to RuntimeException  
4.Add a ConnectionManager, make it easier to integrate with Spring or 3rd party transaction services.  
5.Implemented a tiny declarative transaction service tool "jTinyTX", it's a tiny transaction tool compare to Spring's huge size.    
6.Original functions of DbUtils 100% kept, obviously, because DbUtils is extended from DbUtils, can be looked as a plugin of DbUtils.    
  
jDbPro is developed as core part of jSqlBox project, but it can be used separately, it runs on Java6 or above.  

### How to use in project?  
Add below in pom.xml:
```
   <dependency>  
      <groupId>com.github.drinkjava2</groupId>  
      <artifactId>jdbpro</artifactId>  
      <version>1.7.0.2</version>  <!-- Or latest release -->
   </dependency>
``` 
jDbPro depends on DbUtils, if use Maven will automatically download commons-dbutils-1.7.jar   

### Introduce   
1. jDbPro mixed use normal style, In-line style and template style demo, default if no transaction service configurated, jDbPro works on auto-commit mode.  
query(Connection, String sql, Object... params):   Original DbUtils methods,  need close Connection and catch SQLException  
query(String sql, Object... params):   Original DbUtils methods, need catch SQLException  
nQuery(String sql, Object... params):  new style, no need catch SQLException  
iQuery(String... inlineSQLs):  In-line style  
tQuery(String... sqlTemplate):  SQL Template style  
Below is the demo shows different styles:
``` 
	@Test
	public void executeTest() {
		DbPro dbPro = new DbPro((DataSource) BeanBox.getBean(DataSourceBox.class));
		dbPro.setAllowShowSQL(true);
		User user = new User();
		user.setName("Sam");
		user.setAddress("Canada");

		System.out.println("Example#1: DbUtils old style methods, need close connection and catch SQLException");
		Connection conn = null;
		try {
			conn = dbPro.prepareConnection();
			dbPro.execute(conn, "insert into users (name,address) values(?,?)", "Sam", "Canada");
			dbPro.execute(conn, "update users set name=?, address=?", "Sam", "Canada");
			Assert.assertEquals(1L, dbPro.queryForObject(conn, "select count(*) from users where name=? and address=?",
					"Sam", "Canada"));
			dbPro.execute(conn, "delete from users where name=? or address=?", "Sam", "Canada");
		} catch (SQLException e) {
			e.printStackTrace();
		} finally {
			try {
				dbPro.close(conn);
			} catch (SQLException e) {
				e.printStackTrace();
			}
		}

		System.out.println("Example#2: DbUtils old style methods, need catch SQLException");
		try {
			dbPro.execute("insert into users (name,address) values(?,?)", "Sam", "Canada");
			dbPro.execute("update users set name=?, address=?", "Sam", "Canada");
			Assert.assertEquals(1L,
					dbPro.queryForObject("select count(*) from users where name=? and address=?", "Sam", "Canada"));
			dbPro.execute("delete from users where name=? or address=?", "Sam", "Canada");
		} catch (SQLException e) {
			e.printStackTrace();
		}

		System.out.println("Example#3: nXxxx methods no need catch SQLException");
		dbPro.nExecute("insert into users (name,address) values(?,?)", "Sam", "Canada");
		dbPro.nExecute("update users set name=?, address=?", "Sam", "Canada");
		Assert.assertEquals(1L,
				dbPro.nQueryForObject("select count(*) from users where name=? and address=?", "Sam", "Canada"));
		dbPro.nExecute("delete from users where name=? or address=?", "Sam", "Canada");

		System.out.println("Example#4: iXxxx In-line style methods");
		dbPro.iExecute("insert into users (", //
				" name ,", param0("Sam"), //
				" address ", param("Canada"), //
				") ", valuesQuesions());
		param0("Sam", "Canada");
		dbPro.iExecute("update users set name=?,address=?");
		Assert.assertEquals(1L, dbPro.iQueryForObject("select count(*) from users where name=" + question0("Sam")));
		dbPro.iExecute("delete from users where name=", question0("Sam"), " and address=", question("Canada"));

		System.out.println("Example#5: Another usage of iXxxx inline style");
		dbPro.iExecute("insert into users (", inline0(user, "", ", ") + ") ", valuesQuesions());
		dbPro.iExecute("update users set ", inline0(user, "=?", ", "));
		Assert.assertEquals(1L,
				dbPro.iQueryForObject("select count(*) from users where ", inline0(user, "=?", " and ")));
		dbPro.iExecute(param0(), "delete from users where ", inline(user, "=?", " or "));

		System.out.println("Example#6: tXxxx Sql Template style methods");
		put0("user", user);
		dbPro.tExecute("insert into users (name, address) values(#{user.name},#{user.address})");
		put0("name", "Sam");
		put("addr", "Canada");
		dbPro.tExecute("update users set name=#{name}, address=#{addr}");
		Assert.assertEquals(1L,
				dbPro.tQueryForObject("select count(*) from users where ${col}=#{name} and address=#{addr}",
						put0("name", "Sam"), put("addr", "Canada"), replace("col", "name")));
		dbPro.tExecute("delete from users where name=#{name} or address=#{addr}", put0("name", "Sam"),
				put("addr", "Canada"));
	}
 
```		
 
2. Transaction demo
    jDbPro has no transaction function, so it should supported by 3rd party transaction service tool like Spring.  
	Here is a demo shows use jTransactions as the transaction service. "jTransactions" is a seperated transaction services tool, included 2 implementations: TinyTx and SpringTx. Below demo show at runtime switch use transaction service between TinyTx and SpringTx:  
```
 public class TxDemo {
	private static Class<?> tx = TinyTx.class;
	private static ConnectionManager cm = TinyTxConnectionManager.instance();
	// private static Class<?> tx = SpringTx.class;
	// private static ConnectionManager cm = SpringTxConnectionManager.instance();

	public static class TxBox extends BeanBox {
		{this.setConstructor(tx, BeanBox.getBean(DataSourceBox.class), Connection.TRANSACTION_READ_COMMITTED);
		}
	}

	DbPro dbpro = new DbPro((DataSource) BeanBox.getBean(DataSourceBox.class), cm);

	@AopAround(TxBox.class)
	public void tx_Insert1() {
		dbpro.nExecute("insert into users (id) values(?)", 123);
		Assert.assertEquals(1L, dbpro.nQueryForObject("select count(*) from users"));
	}

	@AopAround(TxBox.class)
	public void tx_Insert2() {
		dbpro.nExecute("insert into users (id) values(?)", 456);
		Assert.assertEquals(2L, dbpro.nQueryForObject("select count(*) from users")); 
		System.out.println(1 / 0);
	}

	@Test
	public void doTest() {
		System.out.println("============Testing: TxDemo============");
		TxDemo tester = BeanBox.getBean(TxDemo.class);
		try {
			dbpro.nExecute("drop table users");
		} catch (Exception e) {
		}
		dbpro.nExecute("create table users (id varchar(40))engine=InnoDB");
		Assert.assertEquals(0L, dbpro.nQueryForObject("select count(*) from users"));
		try {
			tester.tx_Insert1();// this one inserted 1 record
			tester.tx_Insert2();// this one did not insert, roll back
		} catch (Exception e) {
			System.out.println("div/0 exception found, tx_Insert2 should roll back");
		}
		Assert.assertEquals(1L, dbpro.nQueryForObject("select count(*) from users"));
		BeanBox.defaultContext.close();// Release DataSource Pool
	}
}
```
Above are all documents of jDbPro, if have any question please run unit-tests or look over source code.