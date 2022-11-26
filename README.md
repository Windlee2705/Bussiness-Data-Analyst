# Data Analyst Report

# I. Làm sạch dữ liệu

## I. Tạo bảng data_buy

> Bảng data_buy thể hiện người mua hàng, ngày mua và số lượng mua tương ứng. Bảng sẽ bao gồm 3 trường là **visitor_id, noOrders** và **create_date.**
> 

Từ yêu cầu, ta có 2 file dữ liệu là 

- File **data** chứa thông tin thống kê số lượng đơn hàng đã hoàn tất (**order**) theo từng khách hàng (**customer**). Khách hàng là những **visitor_id** có trong file **data**.
- File **visitor information** chứa thông tin lần đầu tiên (**create_date**) khách ghé vào (**visitor**) một trong các địa điểm Online và Offline của công ty **(visitor_source).**

Cả 2 File dữ liệu này đều được xuất ra từ hệ thống báo cáo SAP và cơ sở dữ liệu nội bộ của công ty.

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled.png)

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%201.png)

Bảng data bao gồm cột visitor_id thể hiện mã khách hàng của người mua hàng, cùng với các cột từ 43770 đến 43869 đại diện cho ngày 01/11/2019 đến ngày 08/02/2020. 

Ta sẽ sắp xếp lại bảng data bằng cách dùng hàm melt trong Python để đảo các cột thể hiện ngày từ 01/11/2019 đến 08/02/2020 thành các hàng và có một cột mới thể hiện số lượng mua hàng vào ngày đó

 

```python
df_melted = df.melt(id_vars= df.columns[0], var_name= "create_datenum", value_name= "noOrders")
df_melted
```

Khi đó ta sẽ thu được bảng data sau khi thay đổi như sau:

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%202.png)

Ta thực hiện chuyển đổi các data cột create_datenum từ dạng int sang dạng date

```python
list = df_melted["create_datenum"].unique()
print(list)
print(type(list))
dict = {}
day = datetime.date(2019, 11, 1)
tdelta = datetime.timedelta(days = 1)
for item in list:
    dict[item] = str(day)
    day = day + tdelta

list = []
for i in df_melted.index:
    list.append(dict[df_melted['create_datenum'][i]])
df_melted["create_date"] = list
```

Sau đó xóa cột create_datenum cũ đi, đồng thời xóa các row mà có noOrders=0 đi:

```python
data_buy = data.drop(labels=data.loc[data['noOrders']==0 ].index)
data_buy
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%203.png)

## II. Kiểm tra visitor_id của bảng data có đều nằm trong bảng visitor_information không?

Ta tạo bảng **data_unduplicate** để đếm xem bảng data có bao nhiêu id khác nhau, và kết quả là 39440. Nhưng ta kiểm tra bảng **visitor_information** thì chỉ có 39439 id, như vậy có một id khách hàng có trong bảng data nhưng lại không có thông tin trong bảng data_information. Dễ thấy đó chính là khách hàng có id là “**Khách vãng lai**”

Đồng thời, ta lấy ra các id không có trong bảng **data** nhưng có trong bảng **visitor_information,** thể hiện những người chỉ ghé qua cửa hàng qua các nguồn để đăng ký tài khoản nhưng không mua hàng. 

```python
visitor_only = pd.merge(data_unduplicate, visitor, on="visitor_id", how="right")
visitor_only.loc[visitor_only['create_date_x'].isnull()]
```

Kết quả trả ra bảng có 86964 dữ liệu:

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%204.png)

III. Kiểm tra liệu có id nào có ngày mua hàng trước ngày ghé thăm các nguồn cửa hàng không?

Ta sẽ kiểm tra liệu có id nào mua hàng trước khi biết đến công ty bằng cách tạo bảng **visitor_buy_detail** có trường **first_visit** và **first_buy**. 

Từ bảng **data_buy**, ta tạo bảng **data_buy_firstTime** bằng cách bỏ các id trùng nhau và chỉ giữ id với lần mua hàng đầu tiên: 

```python
# Bỏ duplicate theo visitor_id ở bảng data_buy, chỉ giữ lại data với lần mua hàng đầu tiên : 
data_buy_firstTime = data_buy.drop_duplicates(subset='visitor_id', keep='first')
data_buy_firstTime
# Sắp xếp lại index và sửa tiêu đề cột cho đúng
data_buy_firstTime = data_buy_firstTime.sort_values(by=['visitor_id','create_date'], ascending=[1,1], ignore_index=True)
data_buy_firstTime = data_buy_firstTime.rename({'create_date' : 'first_buy'},axis=1)
data_buy_firstTime
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%205.png)

Từ đó ta tiền hành merge bảng data_buy_firstTime với bảng visitor_information, và thay các giá trị Null ở trường first_buy thành “Không mua hàng”:

```python
# Thêm thông tin lần đầu mua hàng vào bảng visitor
visitor_first_buy = pd.merge(visitor,data_buy_firstTime,how='left', on='visitor_id')
visitor_first_buy = visitor_first_buy.rename({'create_date' : 'first_visit'},axis=1)

visitor_first_buy.loc[visitor_first_buy["first_buy"].isnull(), "first_buy"] = "Không mua hàng"
visitor_first_buy
```

Lấy các row có first_buy khác với “Không mua hàng”, ta sẽ thu được bảng **visitor_buy_detail**

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%206.png)

Từ bảng này ta tiền hành kiểm tra xem liệu có id nào mà first_buy < first_visit hay không:

```python
# Kiểm tra xem có trường hợp nào first_buy < first_visit không
tdelta = datetime.timedelta(days = 0)
for i in visitor_buy_detail.index:
    a_date = datetime.datetime.strptime(visitor_buy_detail['first_buy'][i],"%Y-%m-%d").date()
    b_date = datetime.datetime.strptime(visitor_buy_detail['first_visit'][i],"%Y-%m-%d").date()
    if a_date - b_date < tdelta:
        print("Xuất hiện trường hợp lần đầu mua hàng sớm hơn lần đầu ghé thăm cửa hàng qua các nguồn")
```

Sau khi kiểm tra, ta tháy không có trường hợp nào có lần đầu mua hàng sớm hơn lần đầu ghé thăm cửa hàng qua các nguồn. Như vậy tất cả các khách hàng đều ghé qua cửa hàng qua các nguồn trước khi thực hiện mua hàng lần đầu tiên.

## IV. Tạo bảng conversion_rate

> Để tiến hành phân tích tỷ lệ chuyển đổi khách hàng, ta phải tạo ra bảng conversion_rate bao gồm các trường **Date, visitor_source, First_Visit và First_VisitBuy**. Trong đó **Date** là ngày, **visitor_source** là các nguồn bán hàng của cửa hàng, **First_Visit** là số lượng khách ghé qua cửa hàng ngày hôm đó, **First_VisitBuy** là số lượng khách ghé qua cửa hàng và đồng thời mua hàng.
> 

Từ bảng visitor_buy_detail, lấy ra những Id có first_buy và first_visit bằng nhau: 

```python
first_visitBuy = visitor_buy_detail.loc[visitor_buy_detail["first_buy"]==visitor_buy_detail["first_visit"]]
first_visitBuy
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%207.png)

Ta tiến hành nhóm bảng first_visitBuy theo các trường first_buy và visitor_source, đồng thời bỏ đi các cột không cần thiết 

```python
first_visitBuy_group = first_visitBuy.groupby(["first_buy","visitor_source"]).count().reset_index()
first_visitBuy_group = first_visitBuy_group.reindex(columns = ["first_buy","visitor_source","visitor_id","noOrders","gender","age"])
first_visitBuy_group = first_visitBuy_group.drop(labels=first_visitBuy_group.columns[3:],axis=1).rename({'visitor_id' : 'First_VisitBuy', 'first_buy' : 'Date'},axis=1)
first_visitBuy_group
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%208.png)

Tương tự, từ bảng visitor_information, ta tiến hành nhóm lại và ta sẽ thu được bảng thể hiện số lượng khách hàng ghé qua cửa hàng theo từng ngày: 

```python
first_visit = visitor_informations.groupby(["create_date","visitor_source"]).count().reset_index()
first_visit_group = first_visit.drop(labels=first_visit.columns[3:],axis=1).rename({'visitor_id' : 'First_Visit', 'create_date' : 'Date'},axis=1)
first_visit_group
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%209.png)

Ta tiến hành merge 2 bảng trên sẽ thu được bảng conversion_rate thể hiện tỷ lệ chuyển đổi khách hàng: 

```python
conversion_rate = pd.merge(first_visit_group,first_visitBuy_group,how="outer",on=["Date","visitor_source"])
conversion_rate.loc[conversion_rate["First_VisitBuy"].isnull(),"First_VisitBuy"] = 0
conversion_rate
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%2010.png)

## V. Tạo bảng dataframe thể hiện số thời gian trung bình một người mua hàng lần nữa

> Để thể hiện thời gian trùng bình mua hàng lần nữa của mỗi người, ta cần biết được số lượng mua hàng của người đó cùng với ngày đầu và ngày cuối người đó mua hàng.
> 

Từ bảng data_buyId, ta tạo bảng data_frame bao gồm các khách hàng mà mua hàng ít nhất từ 2 lần đổ lên. 

```python
dataframe = data_buyId.groupby(["visitor_id"]).count().reset_index()
dataframe = dataframe.reindex(columns=["visitor_id","noOrders"]).rename({"noOrders" : "noBuy"},axis=1)
# Xoá các id chỉ mua hàng một lần 
dataframe = dataframe.drop(labels= dataframe.loc[dataframe['noBuy'] == 1].index)
dataframe = dataframe.sort_values(by = 'visitor_id',ignore_index=True)
```

Đồng thời, ta tạo bảng **last_time** thể hiện ngày mua cuối cùng của người đó. Sau khi tạo, ta thu được bảng như sau: 

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%2011.png)

Từ bảng dữ liệu trên, ta lấy last_buy - first_buy ta sẽ thu được khoảng thời gian giữa ngày mua hàng đầu tiên và ngày cuối cùng tính đến thời điểm hiện tại của mỗi khách hàng.

```python
list = []
for i in last_time.index:
    first = datetime.datetime.strptime(last_time.loc[i]['first_buy'],"%Y-%m-%d").date()
    last = datetime.datetime.strptime(last_time.loc[i]['last_buy'],"%Y-%m-%d").date()
    tdelta = last - first
    list.append(int(re.findall('[0-9]+' ,str(tdelta))[0]))
dataframe["time_delta"] = list
dataframe
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%2012.png)

## VI. Tạo bảng data_frame2_group thể hiện tỷ lệ giữ chân khách hàng

> Để thể hiện được tỷ lệ giữ chân khách hàng, ta cần biết được số lượng khách mya hàng của kỳ, số lượng khách mua của kỳ trước, số lượng khách mất đi và số lượng khách mới có thêm được.
> 

Thêm trường Period vào bảng ****data_buyId**** để có thể biết được ngày mua hàng đó ở kỳ mấy. 

```python
value_of_column = []

interval_of_a_period = datetime.timedelta(days = period)
# interval_of_a_period là 30 ngày
period_number = 1
first_date_in_period = datetime.datetime.strptime(data_frame2.loc[0]['create_date'], "%Y-%m-%d").date()

for i in data_frame2.index:
    if datetime.datetime.strptime(data_frame2.loc[i]['create_date'], "%Y-%m-%d").date() - first_date_in_period + datetime.timedelta(days = 1) > interval_of_a_period:
        first_date_in_period = datetime.datetime.strptime(data_frame2.loc[i]['create_date'], "%Y-%m-%d").date()
        period_number += 1
    if datetime.datetime.strptime(data_frame2.loc[len(data_frame2) - 1]['create_date'], "%Y-%m-%d").date() - first_date_in_period + datetime.timedelta(days = 1) < interval_of_a_period:
        break
    value_of_column.append('period' +'_'+ str(period_number))
length = len(value_of_column)
for i in range(0, (len(data_frame2) - length), 1):
    value_of_column.append("Not enough Period")
```

```python
data_frame2['period'] = value_of_column
data_frame2
```

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%2013.png)

> Not enough Period là những ngày của kỳ mới nhất chưa đủ hết kỳ để báo cáo.
> 

Tạo bảng tính lượng khách hàng của mỗi kỳ. Ta tiến hành nhóm lại theo trường **period** và đếm số lượng khách hàng của mỗi kỳ: 

```python
data_frame2_dropDup = data_frame2_period.drop_duplicates(subset=["visitor_id","period"],keep='first')
data_frame2_dropDup= data_frame2_dropDup.drop(labels=data_frame2_dropDup[data_frame2_dropDup["period"]=="Not enough Period"].index)
data_frame2_group = data_frame2_dropDup.groupby(['period']).count().reset_index()
data_frame2_group = data_frame2_group.drop(labels = data_frame2_group.columns[2:],axis=1 ).rename({"index":"noBuy Period"},axis=1)
data_frame2_group
```

Thêm trường **noBuy Period** thể hiện lượng khách mua hàng của kỳ trước: 

```python
import numpy as np
noBuy_Period=np.array(data_frame2_group["noBuy Period"])
arr_last = [0]
for i in data_frame2_group.index : 
    if i>0 : arr_last.append(noBuy_Period[i-1])
data_frame2_group["noBuy last period"] = arr_last
data_frame2_group
```

Tạo trường **noLost in Period** để đếm số lượng khách hàng mà đã mua ở kỳ trước nhưng sang kỳ tiếp thep không mua. 

```python
lost_arr = [0]
for i in range(2,len(data_frame2_group["period"].unique())+1,1) : 
    count =0
    period1 = data_frame2_dropDup.loc[data_frame2_dropDup["period"]=="period_" + str(i-1)]["visitor_id"].unique()
    period2 = data_frame2_dropDup.loc[data_frame2_dropDup["period"]=="period_" + str(i)]["visitor_id"].unique()
    for j in range(len(period1)): 
        if(period1[j] not in period2) : 
            count +=1
    lost_arr.append(count)
data_frame2_group["noLost in Period"] = lost_arr
data_frame2_group
```

Và trường **noBuy new add** thể hiện số lượng khách mới mua hàng ở kỳ này. 

```python
def func_add(x) : 
    return x["noBuy Period"] + x["noLost in Period"] - x["noBuy last period"]
data_frame2_group["noBuy new add"] =  data_frame2_group.apply(func_add, axis = 1)
data_frame2_group
```

Cuối cùng ta thu được bảng thể hiện tỷ lệ giữ chân khách hàng như sau: 

![Untitled](Data%20Analyst%20Report%20f5626bd182254503b8337511eead6559/Untitled%2014.png)

Bảng này thể hiện tỷ lệ giữ chân khách hàng với chu kỳ là 30 days/period. Tương tự với việc tạo ra bảng thể hiện tỷ lệ giữ chân khách hàng với chu kỳ 15days hay 7days, ta thực hiện như trên với biến period=15 hoặc period=7.

******Như vậy, chúng ta đã làm sạch hết các data để thực hiện cho việc trực quan hóa cũng như báo cáo.******

# II. Phân tích dữ liệu theo yêu cầu