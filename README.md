# data_science_salary_prediction
Salary prediction of the jobs in Data Science sector. 

## Dataset Collection

As part of any data science project, data collection is an important step. Since the goal of this project is to determine the salary of data science job positions. We had to scrape the data to get the most updated dataset.  We looked into the different job posting websites such as Linkedin, Indeed and Glassdoor to get the ideal website to scrape the data. We decided to use glassdoor for the scraping since it had the information we were looking for to predict the salary of data science positions. From glassdoor, we can get information like company name, job title, rating, salary, job description, and from the company tab. In addition, we get information like the company’s size, sector, Founding year, Industry, type, and revenue.

With the help of the [article](https://towardsdatascience.com/selenium-tutorial-scraping-glassdoor-com-in-10-minutes-3d0915c6d905) by [Omer Sakarya](https://github.com/arapfaik) I was able to find the scrap the required data from Glassdoor Uk website.
> [Orginal Source Code](https://github.com/arapfaik/scraping-glassdoor-selenium/blob/master/glassdoor%20scraping.ipynb)

  ![img1](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap001.png)
  
  ## Web Scraping
  
  To scrape the data from a website, we can find the web page element by inspecting the webpage source coded. Web page elements reside in a hierarchy. Therefore, we need to point the element needed to scrape in our code. As we can in the example below, the text ‘Data Scientist’ is stored in a class called ‘e1tk4kwz4.’

  ![img3](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap3.png)
  
  Therefore, we write a line of code to scrape the data. Since this text is stored in the class, we find the element by the class name and store it in the variable. 
```python
while not collected_successfully:
                try:
                    company_name = driver.find_element_by_class_name("e1tk4kwz1").text
                    location = driver.find_element_by_class_name("e1tk4kwz5").text
                    job_title = driver.find_element_by_class_name("e1tk4kwz4").text
                    job_description = driver.find_element_by_xpath('.//div[@class="jobDescriptionContent desc"]').text
                    collected_successfully = True
                except:
                    time.sleep(5)
```
One challenge with scraping the glass door is that if we are not signed up, a signup prompt pops up as soon as we click any web page element and block selenium from clicking anywhere else. The way we can bypass this is to click the ‘X’ button to close the popup window.

```python
try:            
            driver.find_element_by_class_name("react-job-listing").click()
        except ElementClickInterceptedException:
            pass

        time.sleep(2)
        
        try:
            driver.find_element_by_css_selector('[alt="Close"]').click()  #clicking to the X.
        except (NoSuchElementException, StaleElementReferenceException) as e:
            pass
```

By scraping, we are generating a dataset of shape 1500 rows and 12 columns. The data set have columns like Job Title, Salary Estimate, Job Description, Rating,  Company Name, Location,  Size, Founded, Type of ownership,  Industry, Sector,  Revenue.  

  ![img4](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap4.png)

- Job Title: Job position offered by the company 
- Salary Estimate: Estimated salary of the position offered in GBP
- Job Description: Description of the job offered
- Rating: Rating of the company
- Company Name: Name of the company
- Location: Location of the company 
- Size: Number of employees in the company
- Founded: Year the company was founded
- Type of ownership: Private or Government or Public Company
- Industry:  Industry in which the company is based on, e.g., Lending, Accounting
- Sector:  Sector, the company, is based on, e.g., Finance, Accounting, Networking
- Revenue:  Annual revenue of the company in USD

## Data Cleaning

Mostly all scraped data are string values, and we find many noises in the data. So we decided to do some feature engineering to derive new columns to performing our prediction. From the Job Title column, we have many in many different job positions. We like to keep only the job positions which are valuable for our prediction. Therefore, we run a loop to find the key job position such as data scientist, data analyst, data engineer and machine learning engineer; else, we will return value na and drop row with na as job titles. Some of this position comes in different levels such as junior data scientist or senior data analyst depending on which salary may vary. Therefore, we made a function to loop over the job title column to find these positions.

```python
def job_t(title):
    if 'data scientist' in title.lower():
        return 'data scientist'
    elif 'analyst' in title.lower():
        return 'data analyst'
    elif 'data engineer' in title.lower():
        return 'data engineer'
    elif 'machine learning' in title.lower() or 'ml' in title.lower():
        return 'machine learning engineer'
    else :
        return 'na'

def seniority(title):
    if 'graduate' in title.lower() or 'jr' in title.lower() or 'junior' in title.lower() or 'Interim' in title.lower() or 'apprenticeship' in title.lower():
        return 'junior'
    if 'lead' in title.lower() or 'sr' in title.lower() or 'senior ' in title.lower() or 'principal' in title.lower() or 'manager' in title.lower() or 'advanced' in title.lower() or 'snr' in title.lower():
        return 'senior'
    else:
        return 'na'
    

df['job_title'] = df['Job Title'].apply(job_t)
df['seniority_level'] = df['Job Title'].apply(seniority)

df = df[df.job_title != 'na']

```
Salary Estimate columns is a string “£33K - £39K (Glassdoor Est.)”, which have Minimum salary and Maximum salary and string “ (Glassdoor Est.)”. However, for our prediction, we need to find the average salary offered for the job posting.  Therefore, we split the Minimum salary and Maximum salary value and find the average salary for the job posting. 

```python
df = df[df['Salary Estimate'] != "-1"] 
Salary = df['Salary Estimate'].apply(lambda x: x.split('(')[0])
Salary_num = Salary.apply(lambda x: x.replace('K','').replace('£','').replace(' ',''))
df['min_salary'] = Salary_num.apply(lambda x: int(x.split('-')[0]))
df['max_salary'] = Salary_num.apply(lambda x: int(x.split('-')[-1]))
df['Avg_salary'] = (df['min_salary'] + df['max_salary'])/2
```

Other factors impacting the salary would be the company’s age and location in which the job is offered. If we subtract the founded year from the current year, we can determine the company’s age. In addition, we have location columns that have a city name and region name. From which, we derive only the city column for our prediction. 

```python
df['city'] = df['Location'].apply(lambda x: x.split(',')[0])
df['age'] = df.Founded.apply(lambda x: x if x<1 else 2021 - x)
```

The skills the job position requires could be a possible factor to determine the possible salary of the job offered. Therefore, we run a loop in Job Description to find the essential skills such as Python, SQL, R, AWS, Tableau, PowerBI, Spark, excel, azure. 

```python
#description
#Finding skill requirement from description
#(Skills: Python, SQL, R, AWS, tableau, PowerBI, Spark, excel, azure)

df['Job Description'] = df['Job Description'].astype(str)

#python
df['py_yn'] = df['Job Description'].apply(lambda x: 1 if 'python' in x.lower() else 0)
#sql
df['sql_yn'] = df['Job Description'].apply(lambda x: 1 if 'sql' in x.lower() else 0)
#R
df['R_yn'] = df['Job Description'].apply(lambda x: 1 if 'r studio' in x.lower() or 'r-studio' in x.lower() else 0)
#AWS
df['aws_yn'] = df['Job Description'].apply(lambda x: 1 if 'aws' in x.lower() else 0)
#tableau
df['tableau_yn'] = df['Job Description'].apply(lambda x: 1 if 'tableau' in x.lower() else 0)
#R
df['pbi_yn'] = df['Job Description'].apply(lambda x: 1 if 'powerbi' in x.lower() or 'power-bi' in x.lower() or 'power bi' in x.lower()else 0)
#spark
df['spark_yn'] = df['Job Description'].apply(lambda x: 1 if 'spark' in x.lower() else 0)
#excel
df['excel_yn'] = df['Job Description'].apply(lambda x: 1 if 'excel' in x.lower() else 0)
#azure
df['azure_yn'] = df['Job Description'].apply(lambda x: 1 if 'azure' in x.lower() else 0)
```

## Exploratory Data Analysis

From the graphs below, we can identify that the “Rating”,  “Avg_salary” seems to follow a normal distribution, and “Age” does not follow this trend. Therefore, normalisation of the data might be required to make use of this data as a predictor.

  ![img5](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap7.png)

From the boxplot below, we identified that the “Age” column seems to have many outliers. Whereas “Rating” only has “-1” as outliers, these companies do not have any rating listed in glassdoor. “Avg_salary” have outliers at  90, which means some companies are paying high salary possible for a senior position. 

  ![img6](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap8.png)

  ![img7](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap9.png)

From the confusion matrix, we found that the “Rating” and “Avg_salary” have some correlations from the confusion matrix. However, it was interesting to notice that “Age” did not correlate with the “Avg_salary”. At the same time, “Rating” has some correlation with “Age”, which means “Age” and “Avg_salary”  are indirectly correlated to each other. From the above graphs, we can say the most job positions are offered in London, and the most are small companies with an employee size of only 1 to 50 employees. 

  ![img8](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap10.png)

From the graphs above, It can be said that most job posting is for a data analyst and quite a few jobs are listing for a senior level post.

This pivot table helps us determine that data engineers are getting the highest annual average salary. So, while considering the last barplot for job titles, we can say that data engineers and machine learning engineers are possibly getting higher pay. However, the number of vacancies for this job title is pretty less when compared with data scientists and data analysts. Most job postings are for the London region, and most job postings are for the data scientist position. Due to the emergence of the pandemic, there have been few job listings that offer remort working. Analyst jobs seem to be more popular for remort job offers.

  ![img9](https://github.com/jijopjames/data_science_salary_prediction/blob/master/images/Scrap11.png)
