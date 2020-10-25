---
title: "Introduction to Machine Learning Systems"
abstract: "This notebook introduces some of the challenges of building machine learning data systems. It will introduce you to concepts around joining of databases together. The storage and manipulation of data is at the core of machine learning systems and data science. The goal of this notebook is to introduce the reader to these concepts, not to authoritatively answer any questions about the state of Nigerian health facilities or Covid19, but it may give you ideas about how to try and do that in your own country."
layout: talk
author:
- given: Eric
  family: Meissner
  url: https://www.linkedin.com/in/meissnereric/
  twitter: meissner_eric_7 
- given: Andrei
  family: Paleyes
  url: https://www.linkedin.com/in/andreipaleyes/
- given: Neil D.
  family: Lawrence
  twitter: lawrennd
  url: http://inverseprobability.com
date: 2020-07-24
ipynb: true
venue: Virtual DSA
transition: None
---

\include{talk-macros.tex}

\slides{\section{AI via ML Systems}

\include{_ai/includes/supply-chain-system.md}
\include{_ai/includes/aws-soa.md}
\include{_ai/includes/dsa-systems.md}
}

\include{_systems/includes/nigeria-health-intro.md}
\include{_systems/includes/nigeria-nmis-installs.md}
\include{_systems/includes/databases-and-joins.md}
\include{_systems/includes/nigeria-nmis-data-systems.md}
\include{_systems/includes/nigeria-nmis-spatial-join.md}
\include{_systems/includes/nigeria-nmis-sqlite.md}
\notes{\subsection{Covid Data}

Now we have the health data, we're going to combine it with [data about COVID-19 cases in Nigeria over time](https://github.com/dsfsi/covid19africa). This data is kindly provided by Africa open COVID-19 data working group, which Elaine Nsoesie has been working with. The data is taken from Twitter, and only goes up until May 2020. 

They provide their data in github. We can access the cases we're interested in from the following URL.

For convenience, we'll load the data into pandas first, but our next step will be to create a new SQLite table containing the data. Then we'll join that table to our existing tables.}

\notes{
\code{covid_data_url = 'https://raw.githubusercontent.com/dsfsi/covid19africa/master/data/line_lists/line-list-nigeria.csv'
covid_data_csv = 'cases.csv'
urllib.request.urlretrieve(covid_data_url, covid_data_csv)
covid_data = pd.read_csv(covid_data_csv)
}}

\notes{As normal, we should inspect our data to check that it contains what we expect. }

\notes{\code{covid_data.head()}}

\notes{And we can get an idea of all the information in the data from looking at the columns.}

\notes{\code{covid_data.columns}}

\notes{Now we convert this CSV file we've downloaded into a new table in the database file. We can do this, again, with the csv-to-sqlite script.}

\notes{\code{!csv-to-sqlite -f cases.csv -t full -o db.sqlite}}

\notes{\subsection{Population Data}

Now we have information about COVID cases, and we have information about how many health centers and how many doctors and nurses there are in each health center. But unless we understand how many people there are in each state, then we cannot make decisions about where they may be problems with the disease. 

If we were running our ride hailing service, we would also need information about how many people there were in different areas, so we could understand what the *demand* for the boda boda rides might be.

To access the number of people we can get population statistics from the [Humanitarian Data Exchange](https://data.humdata.org/).

We also want to have population data for each state in Nigeria, so that we can see attributes like whether there are zones of high health facility density but low population density.}

\notes{\code{pop_url = 'https://data.humdata.org/dataset/a7c3de5e-ff27-4746-99cd-05f2ad9b1066/resource/d9fc551a-b5e4-4bed-9d0d-b047b6961817/download/nga_pop_adm1_2016.csv'
_, msg = urllib.request.urlretrieve(pop_url,'nga_pop_adm1_2016.csv')
pop_data = pd.read_csv('nga_pop_adm1_2016.csv')}}

\notes{\code{pop_data.head()}}

\notes{To do joins with this data, we must first make sure that the columns have the right names. The name should match the same name of the column in our existing data. So we reset the column names, and the name of the index, as follows.}

\notes{\code{pop_data.columns = ['admin1Name_en', 'admin1Pcode', 'admin0Name_en', 'admin0Pcode', 'population']
pop_data = pop_data.set_index('admin1Name_en')}}

\notes{When doing this for real world data, you should also make sure that the names used in the rows are the same across the different data bases. For example, has someone decided to use an abbreviation for 'Federal Capital Territory' and set it as 'FCT'. The computer won't understand these are the same states, and if you do a join with such data you can get duplicate entries or missing entries. This sort of thing happens a lot in real world data and takes a lot of time to sort out. Fortunately, in this case, the data is well curated and we don't have these problems.}

\notes{\subsection{Save to database file}

The next step is to add this new CSV file as an additional table in our SQLite database. This is done using the script as before.}

\notes{\code{pop_data.to_csv('pop_data.csv')}}

\notes{\code{!csv-to-sqlite -f pop_data.csv -t full -o db.sqlite}}

\notes{\subsection{Computing per capita hospitals and COVID}

The Minister of Health in Abuja may be interested in which states are most vulnerable to COVID19. We now have all the information in our SQL data bases to compute what our health center provision is per capita, and what the COVID19 situation is.

To do this, we will use the ```JOIN``` operation from SQL and introduce a new operation called ```GROUPBY```.}

\notes{#### Joining in Pandas

As before, these operations can be done in pandas or GeoPandas. Before we create the SQL commands, we'll show how you can do that in pandas.

In pandas, the equivalent of a database table is a dataframe. So the JOIN operation takes two dataframes and joins them based on the key. The key is that special shared column between the two tables. The place where the 'holes align' so the two databases can be joined together.

In GeoPandas we used an outer join. In an outer join you keep all rows from both tables, even if there is no match on the key. In an inner join, you only keep the rows if the two tables have a matching key.

This is sometimes where problems can creep in. If in one table Abuja's state is encoded as 'FCT' or 'FCT-Abuja', and in another table it's encoded as 'Federal Capital Territory', they won't match and that data wouldn't appear in the joined table.

In simple terms, a JOIN operation takes two tables (or dataframes) and combines them based on some key, in this case the index of the Pandas data frame which is the state name.}

\notes{\code{pop_joined = zones_gdf.join(pop_data['population'], how='inner')}}

\notes{\subsection{GroupBy in Pandas}

Our COVID19 data is in the form of individual cases. But we are interested in total case counts for each state. There is a special data base operation known as ```GROUP BY``` for collecting information about the individual states. The type of information you might want could be a sum, the maximum value, an average, the minimum value. We can use a GroupBy operation in ```pandas``` and SQL to summarize the counts of covid cases in each state. 

A ```GROUPBY``` operation groups rows with the same key (in this case 'province/state') into separate objects, that we can operate on further such as to count the rows in each group, or to sum or take the mean over the values in some column (imagine each case row had the age of the patient, and you were interested in the mean age of patients.)}

\notes{\code{covid_cases_by_state = covid_data.groupby(['province/state']).count()['case_id']}}

\notes{The ```.groupby()``` method on the dataframe has now given us a new data series that contains the total number of covid cases in each state. We can examine it to check we have something sensible.}

\notes{\code{covid_cases_by_state}}

\notes{Now we have this new data series, it can be added to the pandas data frame as a new column.}

\notes{\code{pop_joined['covid_cases_by_state'] = covid_cases_by_state}}

\notes{The spatial join we did on the original data frame to obtain hosp_state_joined introduced a new column, index_right which contains the state of each of the hospitals. Let's have a quick look at it below.}

\notes{\code{hosp_state_joined['index_right']}

\notes{To count the hospitals in each of the states, we first create a grouped series where we've grouped on these states.}

\notes{\code{grouped = hosp_state_joined.groupby('index_right')}}

\notes{This python operation now goes through each of the groups and counts how many hospitals there are in each state. It stores the result in a dictionary. If you're new to Python, then to understand this code you need to understand what a 'dictionary comprehension' is. In this case the dictionary comprehension is being used to create a python dictionary of states and total hospital counts. That's then being converted into a ```pandas``` Data Series and added to the ```pop_joined``` dataframe.}

\notes{\code{counted_groups = {k: len(v) for k, v in grouped.groups.items()}
pop_joined['hosp_state'] = pd.Series(counted_groups)}}

\notes{For convenience, we can now add a new data series to the data frame that contains the per capita information about hospitals. that makes it easy to retrieve later.}

\notes{\code{pop_joined['hosp_per_capita_10k'] = (pop_joined['hosp_state'] * 10000 )/ pop_joined['population']}}

\notes{\subsection{SQL-style}

That's the ```pandas``` approach to doing it. But ```pandas``` itself is inspired by database language, in particular relational databases such as SQL. To do these types of joins at scale, e.g. for our ride hailing app, we need to see how to do these joins in a database.

As before, we'll wrap the underlying SQL commands with a convenient python command. 

What you see below gives the full SQL command. There is a [```SELECT``` command](https://www.w3schools.com/sql/sql_select.asp), which extracts ```FROM``` a particular table. It then completes an [```INNER JOIN```](https://www.w3schools.com/sql/sql_join_inner.asp) using particular columns (```provice/state``` and ```index_right```)}

\helpercode{def join_counts(conn):
    """
    Calculate counts of cases and facilities per state, join results
    """
    cur = conn.cursor()
    cur.execute("""
                SELECT ct.[province/state] as [state], ct.[case_count], ft.[facility_count]
                FROM
                    (SELECT [province/state], COUNT(*) as [case_count] FROM [cases] GROUP BY [province/state]) ct
                INNER JOIN 
                    (SELECT [index_right], COUNT(*) as [facility_count] FROM [facilities] GROUP BY [index_right]) ft
                ON
                    ct.[province/state] = ft.[index_right]
                """)

    rows = cur.fetchall()
    return rows}}

\notes{Now we've created our python wrapper, we can connect to the data base and run our SQL command on the database using the wrapper.}

\notes{\code{conn = create_connection("db.sqlite")}}

\notes{\code{state_cases_hosps = join_counts(conn)}}

\notes{\code{for row in state_cases_hosps:
    print("State {} \t\t Covid Cases {} \t\t Health Facilities {}".format(row[0], row[1], row[2]))}}
	

\notes{\code{base = nigeria.plot(color='white', edgecolor='black', alpha=0, figsize=(11, 11))
pop_joined.plot(ax=base, column='population', edgecolor='black', legend=True)
base.set_title("Population of Nigerian States")}}

\notes{\code{base = nigeria.plot(color='white', edgecolor='black', alpha=0, figsize=(11, 11))
pop_joined.plot(ax=base, column='hosp_per_capita_10k', edgecolor='black', legend=True)
base.set_title("Hospitals Per Capita (10k) of Nigerian States")}}

\notes{\subsection{Exercise}

1. Add a new column the dataframe for covid cases per 10,000 population, in the same way we computed health facilities per 10k capita.

2. Add a new column for covid cases per health facility. 

Do this in both the SQL and the Pandas styles to get a feel for how they differ.}

\notes{\code{# pop_joined['cases_per_capita_10k'] = ???
# pop_joined['cases_per_facility'] = ???}

\notes{\code{base = nigeria.plot(color='white', edgecolor='black', alpha=0, figsize=(11, 11))
pop_joined.plot(ax=base, column='cases_per_capita_10k', edgecolor='black', legend=True)
base.set_title("Covid Cases Per Capita (10k) of Nigerian States")}}

\notes{\code{base = nigeria.plot(color='white', edgecolor='black', alpha=0, figsize=(11, 11))
pop_joined.plot(ax=base, column='covid_cases_by_state', edgecolor='black', legend=True)
base.set_title("Covid Cases by State")}}

\notes{\code{base = nigeria.plot(color='white', edgecolor='black', alpha=0, figsize=(11, 11))
pop_joined.plot(ax=base, column='cases_per_facility', edgecolor='black', legend=True)
base.set_title("Covid Cases per Health Facility")}}

\thanks



\references
