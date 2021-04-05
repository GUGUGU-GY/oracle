# 姓名：高洋 学号：201810414109 班级：软件工程1班

# 实验目的

    掌握分区表的创建方法，掌握各种分区方式的使用场景。

# 实验内容

Oracle有一个开发者角色resource，可以创建表、过程、触发器等对象，但是不能创建视图。本训练要求：
    
    •本实验使用3个表空间：USERS,USERS02,USERS03。在表空间中创建两张表：订单表(orders)与订单详表(order_details)。
    •使用你自己的账号创建本实验的表，表创建在上述3个分区，自定义分区策略。
    •你需要使用system用户给你自己的账号分配上述分区的使用权限。你需要使用system用户给你的用户分配可以查询执行计划的权限。
    •表创建成功后，插入数据，数据能并平均分布到各个分区。每个表的数据都应该大于1万行，对表进行联合查询。
    •写出插入数据的语句和查询数据的语句，并分析语句的执行计划。
    •进行分区与不分区的对比实验。

# 实验步骤
第1步：在主表orders和从表order_details之间建立引用分区 在study用户中创建两个表：orders（订单表）和order_details（订单详表），两个表通过列order_id建立主外键关联。orders表按范围分区进行存储，order_details使用引用分区进行存储。 创建orders表的部分语句是：

    SQL>CREATE TABLESPACE users02 DATAFILE
    '/home/student/你的目录/pdbtest_users02_1.dbf'
      SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED，
    '/home/student/你的目录/pdbtest_users02_2.dbf' 
      SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED
    EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

    SQL>CREATE TABLESPACE users03 DATAFILE
    '/home/student/你的目录/pdbtest_users02_1.dbf'
      SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED，
    '/home/student/你的目录/pdbtest_users02_2.dbf' 
      SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED
    EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

    SQL> CREATE TABLE orders 
    (
     order_id NUMBER(10, 0) NOT NULL 
     , customer_name VARCHAR2(40 BYTE) NOT NULL 
     , customer_tel VARCHAR2(40 BYTE) NOT NULL 
     , order_date DATE NOT NULL 
     , employee_id NUMBER(6, 0) NOT NULL 
    , discount NUMBER(8, 2) DEFAULT 0 
    , trade_receivable NUMBER(8, 2) DEFAULT 0 
    ) 
    TABLESPACE USERS 
    PCTFREE 10 INITRANS 1 
    STORAGE (   BUFFER_POOL DEFAULT ) 
    NOCOMPRESS NOPARALLEL 
    PARTITION BY RANGE (order_date) 
    (
     PARTITION PARTITION_BEFORE_2016 VALUES LESS THAN (
     TO_DATE(' 2016-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 
     'NLS_CALENDAR=GREGORIAN')) 
     NOLOGGING 
     TABLESPACE USERS 
     PCTFREE 10 
     INITRANS 1 
     STORAGE 
    ( 
     INITIAL 8388608 
     NEXT 1048576 
    MINEXTENTS 1 
    MAXEXTENTS UNLIMITED 
    BUFFER_POOL DEFAULT 
    ) 
    NOCOMPRESS NO INMEMORY  
    , PARTITION PARTITION_BEFORE_2017 VALUES LESS THAN (
    TO_DATE(' 2017-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 
    'NLS_CALENDAR=GREGORIAN')) 
    NOLOGGING 
    TABLESPACE USERS02 
    ...
    );

操作截图：
![](创建order表.png)

创建order_details表的部分语句如下：

    CREATE TABLE order_details
    (
    id NUMBER(10, 0) NOT NULL
    , order_id NUMBER(10, 0) NOT NULL
    , product_name VARCHAR2(40 BYTE) NOT NULL
    , product_num NUMBER(8, 2) NOT NULL
    , product_price NUMBER(8, 2) NOT NULL
    , CONSTRAINT order_details_fk1 FOREIGN KEY  (order_id)
    REFERENCES orders  (  order_id   )
    ENABLE
    )
    TABLESPACE USERS
    PCTFREE 10 INITRANS 1
    STORAGE (BUFFER_POOL DEFAULT )
    NOCOMPRESS NOPARALLEL
    PARTITION BY REFERENCE (order_details_fk1);

![](创建order_details表.png)

    declare
      dt date;
      m number(8,2);
      V_EMPLOYEE_ID NUMBER(6);
      v_order_id number(10);
      v_name varchar2(100);
      v_tel varchar2(100);
      v number(10,2);
      v_order_detail_id number;
    begin
      v_order_detail_id:=1;
      delete from order_details;
      delete from orders;
      for i in 1..10000
      loop
        if i mod 6 =0 then
          dt:=to_date('2015-3-2','yyyy-mm-dd')+(i mod 60);  --PARTITION_2015
        elsif i mod 6 =1 then
          dt:=to_date('2016-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2016
        elsif i mod 6 =2 then
          dt:=to_date('2017-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2017
        elsif i mod 6 =3 then
          dt:=to_date('2018-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2018
        elsif i mod 6 =4 then
          dt:=to_date('2019-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2019
        else
          dt:=to_date('2020-3-2','yyyy-mm-dd')+(i mod 60); --PARTITION_2020
        end if;
        V_EMPLOYEE_ID:=CASE I MOD 6 WHEN 0 THEN 11 WHEN 1 THEN 111 WHEN 2 THEN 112
                                WHEN 3 THEN 12 WHEN 4 THEN 121 ELSE 122 END;
        --插入订单
        v_order_id:=i;
        v_name := 'aa'|| 'aa';
        v_name := 'zhang' || i;
        v_tel := '139888883' || i;
        insert /*+append*/ into ORDERS (ORDER_ID,CUSTOMER_NAME,CUSTOMER_TEL,ORDER_DATE,EMPLOYEE_ID,DISCOUNT)
          values (v_order_id,v_name,v_tel,dt,V_EMPLOYEE_ID,dbms_random.value(100,0));
        --插入订单y一个订单包括3个产品
        v:=dbms_random.value(10000,4000);
        v_name:='computer'|| (i mod 3 + 1);
        insert /*+append*/ into ORDER_DETAILS(ID,ORDER_ID,PRODUCT_NAME,PRODUCT_NUM,PRODUCT_PRICE)
          values (v_order_detail_id,v_order_id,v_name,2,v);
        v:=dbms_random.value(1000,50);
        v_name:='paper'|| (i mod 3 + 1);
        v_order_detail_id:=v_order_detail_id+1;
        insert /*+append*/ into ORDER_DETAILS(ID,ORDER_ID,PRODUCT_NAME,PRODUCT_NUM,PRODUCT_PRICE)
          values (v_order_detail_id,v_order_id,v_name,3,v);
        v:=dbms_random.value(9000,2000);
        v_name:='phone'|| (i mod 3 + 1);

        v_order_detail_id:=v_order_detail_id+1;
        insert /*+append*/ into ORDER_DETAILS(ID,ORDER_ID,PRODUCT_NAME,PRODUCT_NUM,PRODUCT_PRICE)
      values (v_order_detail_id,v_order_id,v_name,1,v);
        --在触发器关闭的情况下，需要手工计算每个订单的应收金额：
        select sum(PRODUCT_NUM*PRODUCT_PRICE) into m from ORDER_DETAILS where ORDER_ID=v_order_id;
        if m is null then
         m:=0;
        end if;
        UPDATE ORDERS SET TRADE_RECEIVABLE = m - discount WHERE ORDER_ID=v_order_id;
        IF I MOD 1000 =0 THEN
          commit; --每次提交会加快插入数据的速度
        END IF;
      end loop;
    end;

操作截图：
![](插入订单.png)

第3步:查询orders表：
    
    select count(*) from orders;

操作截图
![](查询orders表.png)


第4步:查询order_details表：

    select count(*) from order_details;

![](查询order_details表.png)

--以system用户运行：
    
    set autotrace on

    select * from your_user.orders where order_date
    between to_date('2017-1-1','yyyy-mm-dd') and to_date('2018-6-1','yyyy-mm-dd');

    select a.ORDER_ID,a.CUSTOMER_NAME,
    b.product_name,b.product_num,b.product_price
    from your_user.orders a,your_user.order_details b where
    a.ORDER_ID=b.order_id and
    a.order_date between to_date('2017-1-1','yyyy-mm-dd') and to_date('2018-6-1','yyyy-mm-dd');
操作结果：
![](6.png)
![](7.png)



# 数据库和表空间占用分析

    当全班同学的实验都做完之后，数据库pdborcl中包含了每个同学的角色和用户。 所有同学的用户都使用表空间users存储表的数据。 表空间中存储了很多相同名称的表mytable和视图myview，但分别属性于不同的用户，不会引起混淆。 随着用户往表中插入数据，表空间的磁盘使用量会增加。

# 查看数据库的使用情况

![](数据库使用情况.png)

    • autoextensible是显示表空间中的数据文件是否自动增加。
    • MAX_MB是指数据文件的最大容量。