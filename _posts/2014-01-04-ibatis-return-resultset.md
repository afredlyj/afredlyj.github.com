---
layout: post
title: ibatis存储过程返回结果集
category: program
---

@[ibatis|db]

####ibatis调用存储过程返回结果集，sqlmap中有两种配置方式：  
 - 在`parameterMap`中设置一个ResultSet，并配置设置resultMap，这时，dao层请求参数map泛型为Map<String, Object>：  

~~~~
	Map<String, Object> map = new HashMap<String, Object>();
		map.put("notify_id", "d");
		map.put("partner_code", "dd");
		map.put("partner_order", "CZ2012071613582027039774723753");
		map.put("orders", "d");
		map.put("pay_result", "OK");
		map.put("amount", "0.01");
		map.put("channel", "8");

		try {
			sqlMap_write.queryForObject("recharge", map);
			Object obj = map.get("out_result");
			System.out.println(obj.getClass());
		} catch (Exception e) {
			e.printStackTrace();
		}

~~~~    

**注意:**这里打印出来的`out_result`类型是`java.util.ArrayList`。  
相应的sqlmap中应该这样配置：  

~~~~
<resultMap class="java.util.HashMap" id="recharge_map">
	<result property="ssoid" column="SSOID"/>
	<result property="callbackUrl" column="CALLBACKURL"/>
	<result property="consumerKey" column="CONSUMERKEY"/>
	<result property="productName" column="PRUDENTNAME"/>
	<result property="productDesc" column="PRUDENTDESC"/>
	<result property="notifyId" column="notifyId"/>
	<result property="out_result" column="out_result"/>
</resultMap>
	
<parameterMap class="java.util.Map" id="recharg_map">
	<parameter property="partner_order" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="orders" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="pay_result" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="partner_code" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="notify_id" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="amount" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="channel" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="out_result" javaType="java.sql.ResultSet" jdbcType="ORACLECURSOR" mode="OUT" resultMap="recharge_map"/>
</parameterMap>
	
<procedure id="recharge" parameterMap="recharg_map">
	{call recharge(?,?,?,?,?,?,?,?)}
</procedure>

~~~~

**注意：**游标的jdbcType为`ORACLECURSOR`，但是我看之前的有些`CURSOR`也没有报错；另外，结果集的`resultMap`直接在`parameterMap`的属性中指定，而节点`procedure`却没有配置`resultMap`。  
 - 返回结果集的另一种写法：  

~~~~
	Map<String, String> result = null;
	try {
		result = (Map<String, String>)sqlMap_write.queryForObject("recharge", map);
		if (log.isDebugEnabled()) {
			if (result != null) {
				for (Map.Entry<String, String> entry : result.entrySet()) {
					log.debug("key:" + entry.getKey() + "; value:" + entry.getValue());
				}
			}
		}
	} catch (Exception e) {
		log.error("回调处理异常,orders=" + map.get("orders"), e);
	}

~~~~  

**注意：**这里的map泛型为Map<String, String>。  
对应的sqlmap配置如下：  

~~~~
<parameterMap class="java.util.Map" id="recharg_map">
	<parameter property="partner_order" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="orders" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="pay_result" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="partner_code" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="notify_id" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="amount" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="channel" javaType="java.lang.String" jdbcType="VARCHAR" mode="IN"/>
	<parameter property="out_result" javaType="java.sql.ResultSet" jdbcType="ORACLECURSOR" mode="OUT"/>
</parameterMap>
	
<resultMap class="java.util.HashMap" id="recharge_map">
	<result property="ssoid" column="SSOID"/>
	<result property="callbackUrl" column="CALLBACKURL"/>
	<result property="consumerKey" column="CONSUMERKEY"/>
	<result property="productName" column="PRUDENTNAME"/>
	<result property="productDesc" column="PRUDENTDESC"/>
	<result property="notifyId" column="notifyId"/>
	<result property="out_result" column="out_result"/>
</resultMap>
	
<procedure id="recharge" parameterMap="recharg_map" resultMap="recharge_map">
	{call recharge(?,?,?,?,?,?,?,?)}
</procedure>

~~~~  

这里在节点`procedure`中配置了`parameterMap`和`resultMap`，另外，`resultMap`的class不能是接口，比如`java.util.Map`，而应该是`java.util.HashMap`，其实，这里配置procedure的结果时，可以不用resultMap，而是直接`resultClass="java.util.HashMap"`，当然也可以是其他的java bean。返回到dao层，这里的类型就是map。

####游标组装结果集
存储过程返回结果集时，有时候结果集的数据并不只是来自一张表，有可能需要将存储过程中的变量也顺带返回，这时候的处理，在mysql和oracle中相差不大，都是直接写在`select`中即可。
 - mysql组装：

~~~~
select  prizename as giftName,imgurl  as giftImgUrl,prizedesc as giftdesc,prizetype as prizeType,id, amount  from  prizeinfo where   grade=p_grade  and osversion=p_osversion  and current_timestamp() between starttime and endtime and awardstatus='ok';
~~~~

其中`awardstatus`是存储过程定义的变量。

 - oracle组装：

~~~~
open my_outCur for 
select callbackurl, ssoid,consumerKey,prudentname,prudentdesc,p_out_result as out_result,to_char(v_count) as notifyId from rechargeinfo where orderno=p_partner_order;
~~~~

其中，`p_out_result`和`v_count`都是定义的变量名。
