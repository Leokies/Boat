# Boat
### In this project I show you how data analysis helped me to find my dream sailboat.

- [ASK](#ASK)
	- [Anaysis task](#Anaysis_task)
	- [Establish_metrics](#Establish_metrics)
- [Prepare](#Prepare)	
- [Process](#Process)
- [Analysis](#Analysis)
 	- [Python](#Python)



<a name="ASK"></a>
## ASK phase:

<a name="Anaysis_task"></a>
### Anaysis task:
Find a sailboat that is suitable to live on, is ocean capable and fits in my budget.
As a student, I have a limited budget. Therefore, the main consideration will be the price of the boat. My budget is 30000 €.
From my own sailing experience I am looking for a minimum boat length of 9 meters. This is the smallest size I can live with comfortably on the boat with my girlfriend and feel comfortable even on long ocean passages.

<a name="Establish_metrics"></a>
### Establish metrics:
-Investigation if there is a correlation between the price and boat length, boat width , year of construction?

-Which countries have the most boats whith a suitble price and budget boat length? 

<a name="Prepare"></a>
## Prepare phase:
We Download the [boat_sales](https://www.kaggle.com/datasets/karthikbhandary2/boat-sales) dataset from [Kaggle](https://www.kaggle.com/datasets/karthikbhandary2/boat-sales) and import it to SQL.


![image](https://user-images.githubusercontent.com/94685505/224360118-23534327-21d6-4048-81cb-f86b9d7cf23a.png)


First of all we get an overview of the data. The data consist out of 9892 rows and 10 columns. 
With the columns: Price in different currencies, Boat type, Manufacturer, Type, Year Build, Length, Width, Location and Number of views last 7 days.
All values are stored as varchar(50).

We create a new table `boat_data` that contains only the required columns. Furthermore, we adjust the data type of each column.

```
CREATE TABLE boat_data
	price varchar(50),
	boat_type varchar(50),
	built int,
	boat_length decimal(10,2),
	boat_width decimal(10,2),
	material varchar(50),
	loc varchar(100)
	)
```
```
INSERT INTO boat_data (price, boat_type,condition, built, boat_length, boat_width, material, loc)
SELECT Price as price,
	[Boat Type] as boat_type,
	[Type] as condition,
	CAST([Length] as decimal(10,2) as boath_length,
	CAST([Width] as decimal(10,2) as boat_width,
	Material as material,
	[Location] as loc
FROM boat_raw
```
<a name="Process"></a>
## Process phase

Now let's take a closer look at the data to detect irregularities and errors.
The first column shows the boat prices in different currencies. We need to correct this if we want to compare the prices.

We create two new columns `currency` and `price_euro`. Then we split the column with the LEFT and SUBSTRING function. Furthermore, we change the datatype of the column `amount` as a decimal.

```
ALTER TABLE boat_data
ADD currency VARCHAR(3)

UPDATE boat_data
SET currency= LEFT(price,3)
```
```
ALTER TABLE boat_data
ADD price_euro decimal(10,2)

UPDATE boat_data
SET price_euro= CAST(SUBSTRING(price, 5, LEN(price)-4) as decimal(10,2)
```
Now we can convert all prices to euro. We do this with the `CASE` `WHEN` function.

```
UPDATE boat_data
SET price_euro=
	CASE	
		WHEN currency LIKE 'EUR' THEN price_euro
		WHEN currency LIKE 'CHF' THEN price_euro amount * 1.01
		WHEN currency LIKE 'DKK' THEN price_euro amount * 0.13
		WHEN currency LIKE '?œ ' THEN price_euro amount * 0.89
		ELSE NULL
	END 
FROM boat_data
```

Next, we filter the boat types to obtain only sailboats. By the DISTINCT command we see that there are 126 different boat types.
Then we apply a filter to the “boat_type” column to filter out only matching boats.

```
DELETE FROM boat_data
WHERE boat_data NOT IN ('Cabin Boat','Hardtop','Sport Boat','Classic','Motorsailer','Deck Boat','Classic','Sport Boat','Offshore Boat')
```
Next, we consider the "conditions" column. This consists of non-uniform sentences, which include whether the boat is new or used.
To maintain a uniform form, we use the CASE WHEN. We check whether "new" or "used" occurs somewhere in the cell and then output only the respective word.

```
UPDATE boat_data
SET conditon =
		CASE
			WHEN condition LIKE '%new%' THEN 'NEW'
			WHEN condition LIKE '%used%' THEN 'Used'
		END;
```

The column `material` has some empty cells. We mark these as `Unknown`. 
With `TRIM(material)= '';` we check if the cell consist out of whitespace characters which should be treated as blanks.

```
UPDATE boat_data
SET material ='Uknown' WHERE material IS NULL or TRIM(material)= '';
```

The last step is the "loc" column. The column consists of inconsistent sentences which however contain country names. We only want the country name.
In order to do so we use a combination of the LEFT, NULLIF and CHARDINDEX function.
```
UPDATE boat_data
SET loc= LEFT(loc, NULLIF(CHARINDEX('?', loc), 0)-1)
WHERE CHARINDEX('?',loc) >0;
```
`CHARINDEX('?', loc)` returns the position of the first occurrence of the character '?' in the `loc` column. If the character is not found, 0 is returned.

The `NULLIF` function is used to return NULL if `CHARINDEX('?', loc)` returns 0. This is important because if`CHARINDEX('?', loc)` returns 0 and we subtract 1 from it, we get an error because the second argument of the `LEFT` function should be greater than or equal to 1.

So if `CHARINDEX('?', loc)` returns a value greater than 0, the `NULLIF` function returns this value minus 1. Otherwise, it returns NULL, which causes the `LEFT` function to return NULL as well.

Great now our dataset looks good. It is clear and contains the information we need and has the right data type. We are ready for the analysis. 

The only thing we are still missing are coordinates for our country so that we can later display our results as a map. 
For this we load a dataset with countries and their coordinates. Then we join the two datasets with a LEFT OUTER JOIN.

```
SELECT condition,built,boat_length,boat_width,material,loc,price_euro, latitude,longitude
FROM Portfolio.dbo.boat_data
LEFT OUTER JOIN Portfolio.dbo.countries_lat_lon
	ON boat_data.loc = countries_lat_lon.name
```
Now we add `latitude` and `longitude` to the boat_data table.

```
ALTER TABLE Portfolio.dbo.boat_data
ADD latitude FLOAT,
    longitude FLOAT;
```
```
UPDATE Portfolio.dbo.boat_data
SET latitude = countries_lat_lon.latitude,
    longitude = countries_lat_lon.longitude
FROM Portfolio.dbo.countries_lat_lon
WHERE boat_data.loc = countries_lat_lon.name;
```

Final table: 

![image](https://user-images.githubusercontent.com/94685505/224407357-7b209e58-9914-4f44-822c-b53625b29825.png)

<a name="Analysis"></a>
## Analysis phase

Finally we can start analyzing our dataset. 

<a name="Python"></a>
### Python

Our first task is to identify the main factors which influence the price. 
For this purpose, we calculate a correlation matrix between the price, boat length, boat width and year of construction with python.

First we import our dataset with pandas into a dataframe `df`
```
import pandas as pd

df= pd.read_csv('boat.csv')
```
![image](https://user-images.githubusercontent.com/94685505/224372463-21b38244-fbd6-40a4-8d3b-fdd5ada3578b.png)

```
corr_matrix= df.corr()

print(corr_matrix)
```
![image](https://user-images.githubusercontent.com/94685505/224372941-c584e349-e1c9-4028-9e7e-67487ba3f41a.png)

The matrix shows a moderate to strong correlation between `price` and `boat length` (0.67) as well as `price` and `boat width` (0.66). Furthermore, we can see that the `price` is only weakly related to the year of construction`built` (0.18). Besides, we can notice that there is a strong correlation between `boots length` and `boat width` (0.81).

We can conclude that the price is especially related to the boot length. 

This allows us to refine our boot search further.Since we have to pay special attention to our budget, it is advisable to consider small boats. So we are looking for boats around 9m to maximum 11m.


Ok now let's see which country has the most suitable boats for us. We are looking for boats that cost 30000 euros or less and are between 9 and 11 meters long.
```
import pandas as pd

# filter the data to keep boats with price less than 30000 and length between 9 and 10 meters
filtered_data = df[(df['price_euro'] < 30000) & (df['boat_length'] >= 9) & (df['boat_length'] <= 11)]

# group the filtered data by country and count the number of boats in each group
counts_by_country = filtered_data.groupby('loc')['price_euro'].count()

print(counts_by_country)
```
![image](https://user-images.githubusercontent.com/94685505/224407546-ee7d26b4-d07b-425d-9130-07bd0eec7bf6.png)





Finally, we would like to present this information in a pie chart.
```
import matplotlib.pyplot as plt

filtered_data = df[(df['price_euro'] <= 30000) & (df['boat_length'] >= 9) & (df['boat_length'] <= 11)]

counts_by_country = filtered_data.groupby('loc')['price_euro'].count()

fig, ax = plt.subplots(figsize=(8, 8))

ax.pie(counts_by_country.values, labels=counts_by_country.index, autopct='%1.1f%%', startangle=90)

ax.set_title('Boats by Country')

plt.show()
```






![image](https://user-images.githubusercontent.com/94685505/224429020-65ca0a78-1639-4ae9-9cf2-a94d31f91cda.png)





### With this information I have made myself on the search for Italy. Three months later I actually found my dream sailboat in Rome......




![WhatsApp Image 2023-03-10 at 20 07 23](https://user-images.githubusercontent.com/94685505/224429581-d737d896-8f5a-4705-8465-5ae1d36cf650.jpeg)























		



