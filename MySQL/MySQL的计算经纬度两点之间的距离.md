# MySQL的计算经纬度两点之间的距离

## 1.使用经纬度原理计算

```mysql
SELECT
	round(
		6378.138 * 2 * asin(
			sqrt(
				pow( sin(( 30.657462 #A点的纬度

   * pi()/ 180- 30.431714 #B点的纬度
        * pi()/ 180 )/ 2 ), 2 )+ cos( 30.657462 #A点的纬度
             * pi()/ 180 ) * cos( 30.431714 #B点的纬度
             * pi()/ 180 )* pow( sin(( 104.0665460 #A点的经度
                  * pi()/ 180- 104.072704 #B点的经度
     * pi()/ 180 )/ 2 ), 2 )))* 1000);
```

![image-20220529175445091](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022image-20220529175445091.png)

## 2.使用st_distance_sphere函数

```mysql
SELECT round(st_distance_sphere(point(104.0665460, 30.657462), point(104.072704, 30.431714)));
```

![image-20220529175626056](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022image-20220529175626056.png)

## 3.使用st_distance函数

```mysql
SELECT ROUND(st_distance(point(104.0665460, 30.657462), point(104.072704, 30.431714)) * 111195);
```

![image-20220529175732346](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022image-20220529175732346.png)



## 4.计算两点之间距离范围的数据

```mysql
select t.num,t.city,t.wgs84_lng,t.wgs84_lat,
  TRUNCATE(st_distance_sphere(point (113.8064049, 22.7300434),point(t.wgs84_lng,t.wgs84_lat)),2) as distance
	from moran_point t
	where t.city = '深圳' 
	HAVING distance > 0 and distance < 3500
	ORDER BY distance;
	
```