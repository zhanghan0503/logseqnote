- 流程：
	- --temp_table表存储解析ODS_COL
	  ---step 1、插入ODS_COL、YWC_COL、STD到DC_DATA_MAPPING
	  insert into DC_DATA_MAPPING (id,YWC_DB,YWC_TAB,YWC_COL,ODS_TAB,ODS_COL,STD_TAB,STD_COL,STD_STDCODE,DWD_TAB,DWD_COL,DWD_STDCODE,DWS_TAB,DWS_COL,DWS_STDCODE,ADS_TAB,ADS_COL)
	  SELECT 
	      TO_CHAR(TO_NUMBER((SELECT MAX(TO_NUMBER(id)) FROM DC_DATA_MAPPING), '999999')+ ROWNUM) as id,
	      SUBSTR(b.ODS_TABLE_NAME,INSTR(b.ODS_TABLE_NAME, '_', 1, 1) + 1, INSTR(b.ODS_TABLE_NAME, '_', 1, 2) - INSTR(b.ODS_TABLE_NAME, '_', 1, 1) - 1) AS YWC_DB,
	      SUBSTR(b.ODS_TABLE_NAME, INSTR(b.ODS_TABLE_NAME, '_', 1, 2) + 1) AS YWC_TAB,
	      a.ODS_DES as YWC_COL, 
	      b.ODS_TABLE_NAME as ODS_TAB, a.ODS_DES as ODS_COL, 
	      b.STD_TABLE_NAME as STD_TAB, b.STD_FIELD_NAME as STD_COL, b.STD_FIELD_NOTE as STD_STDCODE,
	      null,null,null,null,null,null,null,null
	  FROM 
	      META_STD_FIELD_MAP b
	  JOIN 
	       temp_table a ON a.STD_TABLE_NAME = b.STD_TABLE_NAME and a.STD_FIELD_NAME = b.STD_FIELD_NAME and a.ODS_TABLE_NAME = 'ODS_HAFOIS_PMLSHT'
	  ---step2、DWD插入
	  -- 1: 执行 SELECT 操作，新建临时表temp_table_DWD
	  CREATE TABLE temp_table_DWD AS
	  SELECT 
	      a.id, a.YWC_DB, a.YWC_TAB, a.YWC_COL, a.ODS_TAB, a.ODS_COL, a.STD_TAB, a.STD_COL, a.STD_STDCODE, 
	      b.DWD_TABLE_NAME as DWD_TAB, b.DWD_FIELD_NAME as DWD_COL, b.DWD_FIELD_NOTE as DWD_STDCODE, a.DWS_TAB, a.DWS_COL, a.DWS_STDCODE, a.ADS_TAB, a.ADS_COL
	  FROM 
	      DC_DATA_MAPPING a 
	  JOIN 
	      META_DWD_FIELD_MAP b ON a.STD_TAB = b.STD_TABLE_NAME AND a.STD_COL = b.STD_FIELD_NAME WHERE a.ODS_TAB='ODS_HAFOIS_PMLSHT'
	  --- 2：修改相同id
	  UPDATE temp_table_DWD a
	  SET a.id = a.id || '_' || a.DWD_TAB || '_' || a.DWD_COL
	  WHERE EXISTS (
	      SELECT 1
	      FROM temp_table_DWD b
	      WHERE b.id = a.id
	      AND a.rowid > (
	          SELECT MIN(c.rowid)
	          FROM temp_table_DWD c
	          WHERE c.id = b.id
	      )
	  );
	  select * from temp_table_DWD
	  --- 3：修改DC_DATA_MAPPING里面的数据
	  UPDATE DC_DATA_MAPPING dst
	  SET 
	      (
	          dst.YWC_DB, dst.YWC_TAB, dst.YWC_COL,
	          dst.ODS_TAB, dst.ODS_COL,
	          dst.STD_TAB,dst.STD_COL, dst.STD_STDCODE, 
	          dst.DWD_TAB,dst.DWD_COL, dst.DWD_STDCODE,
	          dst.DWS_TAB,dst.DWS_COL,dst.DWS_STDCODE,
	          dst.ADS_TAB,dst.ADS_COL
	      ) = (
	          SELECT 
	              MAX(src.YWC_DB), MAX(src.YWC_TAB), MAX(src.YWC_COL),
	              MAX(src.ODS_TAB), MAX(src.ODS_COL), 
	              MAX(src.STD_TAB),MAX(src.STD_COL), MAX(src.STD_STDCODE),
	              MAX(src.DWD_TAB),MAX(src.DWD_COL), MAX(src.DWD_STDCODE),
	              MAX(src.DWS_TAB),MAX(src.DWS_COL),MAX(src.DWS_STDCODE),
	              MAX(src.ADS_TAB),MAX(src.ADS_COL)
	          FROM 
	              temp_table_DWD src
	          WHERE 
	              dst.id = src.id
	          GROUP BY 
	              src.id
	      )
	  WHERE dst.id IN (SELECT id FROM temp_table_DWD);
	  --- 4：执行插入 DC_DATA_MAPPING表中数据
	  INSERT INTO DC_DATA_MAPPING (id, YWC_DB, YWC_TAB, YWC_COL, ODS_TAB, ODS_COL, STD_TAB, STD_COL, STD_STDCODE, DWD_TAB, DWD_COL, DWD_STDCODE, DWS_TAB, DWS_COL, DWS_STDCODE, ADS_TAB, ADS_COL)
	  SELECT 
	      TO_CHAR(TO_NUMBER((SELECT MAX(TO_NUMBER(id)) FROM DC_DATA_MAPPING), '999999')) + 1 +ROWNUM,
	      YWC_DB, YWC_TAB, YWC_COL,
	      ODS_TAB, ODS_COL,
	      STD_TAB, STD_COL, STD_STDCODE, 
	      DWD_TAB, DWD_COL, DWD_STDCODE,
	      DWS_TAB, DWS_COL, DWS_STDCODE,
	      ADS_TAB, ADS_COL
	  FROM temp_table_DWD
	  WHERE id NOT IN (SELECT id FROM DC_DATA_MAPPING);
	  ---step3、DWS插入
	  -- 1: 执行 SELECT 操作，创建临时表temp_table_DWS
	  CREATE TABLE temp_table_DWS AS 
	  SELECT 
	      a.id, 
	      a.YWC_DB, a.YWC_TAB, a.YWC_COL, a.ODS_TAB, a.ODS_COL, a.STD_TAB, a.STD_COL, a.STD_STDCODE, 
	      a.DWD_TAB, a.DWD_COL, a.DWD_STDCODE, 
	      'DWS_ZB_' || b.ZBBM AS DWS_TAB, b.COLUMN_NAME AS DWS_COL, b.COLUMN_NOTE AS DWS_STDCODE, 
	      a.ADS_TAB, a.ADS_COL
	  FROM 
	      DC_DATA_MAPPING a
	  JOIN 
	      META_DWS_ZB b ON a.DWD_TAB = b.TABLE_NAME AND a.DWD_COL = b.COLUMN_NAME WHERE a.ODS_TAB='ODS_HAFOIS_PMLSHT'
	  ORDER BY 
	      a.id, DWS_TAB, DWS_COL;
	  -- 2：创建表后，执行以下更新语句在id后面拼接字段
	  UPDATE temp_table_DWS a
	  SET a.id = a.id || '_' || a.DWD_TAB || '_' || a.DWD_COL
	  WHERE EXISTS (
	      SELECT 1
	      FROM temp_table_DWS b
	      WHERE b.id = a.id
	      AND a.rowid > (
	          SELECT MIN(c.rowid)
	          FROM temp_table_DWS c
	          WHERE c.id = b.id
	      )
	  );
	  select * from temp_table_DWS
	  -- 3：执行修改，修改DC_DATA_MAPPING表中数据
	  UPDATE DC_DATA_MAPPING dst
	  SET 
	      (
	          dst.YWC_DB, dst.YWC_TAB, dst.YWC_COL,
	          dst.ODS_TAB, dst.ODS_COL,
	          dst.STD_TAB,dst.STD_COL, dst.STD_STDCODE, 
	          dst.DWD_TAB,dst.DWD_COL, dst.DWD_STDCODE,
	          dst.DWS_TAB,dst.DWS_COL,dst.DWS_STDCODE,
	          dst.ADS_TAB,dst.ADS_COL
	      ) = (
	          SELECT 
	              MAX(src.YWC_DB), MAX(src.YWC_TAB), MAX(src.YWC_COL),
	              MAX(src.ODS_TAB), MAX(src.ODS_COL), 
	              MAX(src.STD_TAB),MAX(src.STD_COL), MAX(src.STD_STDCODE),
	              MAX(src.DWD_TAB),MAX(src.DWD_COL), MAX(src.DWD_STDCODE),
	              MAX(src.DWS_TAB),MAX(src.DWS_COL),MAX(src.DWS_STDCODE),
	              MAX(src.ADS_TAB),MAX(src.ADS_COL)
	          FROM 
	              temp_table_DWS src
	          WHERE 
	              dst.id = src.id
	          GROUP BY 
	              src.id
	      )
	  WHERE dst.id IN (SELECT id FROM temp_table_DWS);
	  --- 4：执行插入DC_DATA_MAPPING表中数据
	  INSERT INTO DC_DATA_MAPPING (id, YWC_DB, YWC_TAB, YWC_COL, ODS_TAB, ODS_COL, STD_TAB, STD_COL, STD_STDCODE, DWD_TAB, DWD_COL, DWD_STDCODE, DWS_TAB, DWS_COL, DWS_STDCODE, ADS_TAB, ADS_COL)
	  SELECT 
	      TO_CHAR(TO_NUMBER((SELECT MAX(TO_NUMBER(id)) FROM DC_DATA_MAPPING), '999999')) + 1 +ROWNUM,
	      YWC_DB, YWC_TAB, YWC_COL,
	      ODS_TAB, ODS_COL,
	      STD_TAB, STD_COL, STD_STDCODE, 
	      DWD_TAB, DWD_COL, DWD_STDCODE,
	      DWS_TAB, DWS_COL, DWS_STDCODE,
	      ADS_TAB, ADS_COL
	  FROM temp_table_DWS
	  WHERE id NOT IN (SELECT id FROM DC_DATA_MAPPING);
	  -- 删除临时表
	  DROP TABLE temp_table_DWS;
	  -- step5、ADS插入
	  select a.*,b.* from dc_data_mapping a ,META_ADS_ZB b where SUBSTR(a.DWS_TAB, INSTR(a.DWS_TAB, '_', 1, 2) + 1) = b.ZBBM
	  -- 1: 执行 SELECT 操作，创建临时表temp_table_ADS
	  CREATE TABLE temp_table_ADS AS
	  SELECT distinct
	      a.id, 
	      a.YWC_DB, a.YWC_TAB, a.YWC_COL, a.ODS_TAB, a.ODS_COL, a.STD_TAB, a.STD_COL, a.STD_STDCODE, 
	      a.DWD_TAB, a.DWD_COL, a.DWD_STDCODE, 
	      a.DWS_TAB, a.DWS_COL, a.DWS_STDCODE, 
	      b.TABLE_NAME AS ADS_TAB, b.COLUMN_NAME AS ADS_COL
	  FROM 
	      DC_DATA_MAPPING a 
	  JOIN 
	      META_ADS_ZB b ON SUBSTR(a.DWS_TAB, INSTR(a.DWS_TAB, '_', 1, 2) + 1) = b.ZBBM  WHERE a.ODS_TAB='ODS_HAFOIS_PMLSHT'
	  ORDER BY 
	      a.id, ADS_TAB, ADS_COL;
	  -- 2：创建表后，执行以下更新语句在id后面拼接字段
	  UPDATE temp_table_ADS a
	  SET a.id = a.id || '_' || a.ADS_TAB || '_' || a.ADS_COL
	  WHERE EXISTS (
	      SELECT 1
	      FROM temp_table_ADS b
	      WHERE b.id = a.id
	      AND a.rowid > (
	          SELECT MIN(c.rowid)
	          FROM temp_table_ADS c
	          WHERE c.id = b.id
	      )
	  );
	  select * from temp_table_ADS
	  -- 3：执行修改，DC_DATA_MAPPING表中数据
	  UPDATE DC_DATA_MAPPING dst
	  SET 
	      (
	          dst.YWC_DB, dst.YWC_TAB, dst.YWC_COL,
	          dst.ODS_TAB, dst.ODS_COL,
	          dst.STD_TAB,dst.STD_COL, dst.STD_STDCODE, 
	          dst.DWD_TAB,dst.DWD_COL, dst.DWD_STDCODE,
	          dst.DWS_TAB,dst.DWS_COL,dst.DWS_STDCODE,
	          dst.ADS_TAB,dst.ADS_COL
	      ) = (
	          SELECT 
	              MAX(src.YWC_DB), MAX(src.YWC_TAB), MAX(src.YWC_COL),
	              MAX(src.ODS_TAB), MAX(src.ODS_COL), 
	              MAX(src.STD_TAB),MAX(src.STD_COL), MAX(src.STD_STDCODE),
	              MAX(src.DWD_TAB),MAX(src.DWD_COL), MAX(src.DWD_STDCODE),
	              MAX(src.DWS_TAB),MAX(src.DWS_COL),MAX(src.DWS_STDCODE),
	              MAX(src.ADS_TAB),MAX(src.ADS_COL)
	          FROM 
	              temp_table_ADS src
	          WHERE 
	              dst.id = src.id
	          GROUP BY 
	              src.id
	      )
	  WHERE dst.id IN (SELECT id FROM temp_table_ADS);
	  --- 4：执行插入DC_DATA_MAPPING表中数据
	  INSERT INTO DC_DATA_MAPPING (id, YWC_DB, YWC_TAB, YWC_COL, ODS_TAB, ODS_COL, STD_TAB, STD_COL, STD_STDCODE, DWD_TAB, DWD_COL, DWD_STDCODE, DWS_TAB, DWS_COL, DWS_STDCODE, ADS_TAB, ADS_COL)
	  select 
	      '501',
	      YWC_DB, YWC_TAB, YWC_COL,
	      ODS_TAB, ODS_COL,
	      STD_TAB, STD_COL, STD_STDCODE, 
	      DWD_TAB, DWD_COL, DWD_STDCODE,
	      DWS_TAB, DWS_COL, DWS_STDCODE,
	      ADS_TAB, ADS_COL
	      from temp_table_ADS where id ='470_ADS_SD_YWGLLHJY_LJLHJYFDJE'
	  SELECT 
	      TO_CHAR(TO_NUMBER((SELECT MAX(TO_NUMBER(id)) FROM DC_DATA_MAPPING), '999999')) + 1 +ROWNUM,
	      YWC_DB, YWC_TAB, YWC_COL,
	      ODS_TAB, ODS_COL,
	      STD_TAB, STD_COL, STD_STDCODE, 
	      DWD_TAB, DWD_COL, DWD_STDCODE,
	      DWS_TAB, DWS_COL, DWS_STDCODE,
	      ADS_TAB, ADS_COL
	  FROM temp_table_ADS
	  WHERE id NOT IN (SELECT id FROM DC_DATA_MAPPING);
	  -- 删除临时表
	  DROP TABLE temp_table_DWD;
	  DROP TABLE temp_table_DWS;
	  DROP TABLE temp_table_ADS;
	  select * from temp_table_ADS