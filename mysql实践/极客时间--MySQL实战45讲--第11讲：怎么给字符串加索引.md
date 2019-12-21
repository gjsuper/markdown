# 极客时间--MySQL实战45讲--第11讲：怎么给字符串加索引

* 前缀索引
    - 节约索引空间
    - 使用前缀索引，每次都会找到匹配的值，都会回主索引去查
    - 增加扫描行数
    - 使用前缀索引就不能使用使用覆盖索引对查询性能的优化了，因为系统并不能确定前缀索引的定义是否截断了完整信息，还是要回主索引再查一下
- 使用hash字段：在表上在建一个整数字段，来保存这个字符串的校验码，同时在这个字段上加索引

        mysql> select field_list from user where crc = crc32（input_id_card） and id_card = input_id_card
- 倒序存储：使用前缀索引时，有可能前缀的区分度不高，如果后缀的区分度比较高的话，可以使用后缀索引的方式，将字符串倒序存储，其中索引时input_id_card_string后六位的倒序

        mysql> select field_list from t where id_card = reverse('input_id_card_string');
