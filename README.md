# Asayake Taiko Tryouts Attendance Tracker
The program `tryouts_attendance.exe` is designed easily keep track of tryoutee attendance and can be used each fall quarter with a specific Google Form setup. Program written by Matthew Alegrado, Internal Director 2024-25, discord @ gaiiuss.

To download this, press the green **Code** button and click "Download ZIP". Then extract that onto your computer and follow the steps in the next section.

## Setup
Each Google Form should have an automatic email address collection that retrieves the school email of each tryoutee. This means that your own UCSD email needs to be the owner of the Google Form, not the Internal account. The email shouldn't be an entry in the form (as people tend to misspell the email and screw up results) but the "Verified" option for the "Collect email addresses" setting in the form. 

#### Column Names
The names of the first two columns in the Google Sheet linked to each attendance form should be "Timestamp" and "Email Address". Identifiers such as the person's name and phone number should stay the same between forms (whatever name you choose must be written in `parameters.json`, see "Parameters"). Phone numbers aren't required for the program to run before all data from tryouts is collected.

#### Attendance Spreadsheets
There should be one attendance sheet from each week that must be converted to a .csv file, which can be done by selecting `File > Download > Comma seperated value (.csv)` if on Google Sheets, and then placing it into a folder called `inputs` in the current directory. Likewise, make sure that no other files are in that folder. 

#### Parameters
There are parameters that might need to be changed based on tryouts requirements, i.e. length of tryouts, how many days the tryoutees can miss and need to make up, etc. Most importantly, change the dates `week_1_main` and `week_1_makeup`, corresponding to the days of the first official tryout and the first official makeup, assuming they each occur once a week. 

### Running the Program
Depending on what the variable `week_num` is in `parameters.json`, if it is the last week of tryouts, the output will write to `week_5_performers.xlsx`. This will list all the performers and their information along with if they are deemed eligible based on attendance.

Otherwise, the output will write to `week_X_attendance`, where X is the given `week_num`. Here, it will display the people that attended this week's tryouts and their previous history.

The script will output to .xlsx files, specifically `tryoutees.xlsx` and the one listed in the output. The former is a list of all the tryoutees.

## Notes
Since this program relies on the timestamp of attendance submissions, make sure that no one fills out the attendance on a different day by accident (say, if they forget to do it that day). 

If for whatever reason Asa stops doing Sunday as the tryout day, note that each reference to Sunday is just whichever day the tryouts are each week.

Any attendance mishaps that need to be amended manually should be done so in the original attendance spreadsheet, filled out in the same manner as other entries with the correct timestamp formatting (can copy and paste from another cell). This will ensure the program treats it the same as the other proper entries.


```python
import numpy as np
import pandas as pd
import glob
import re
from datetime import datetime, timedelta
import json
import ast
import openpyxl
import atexit

import warnings
warnings.filterwarnings("ignore")
```


```python
# Reading parameters and dates from 'parameters.json'
dates_string = []
with open('parameters.json') as json_file:
    # Parameters
    data = json.load(json_file)
    num_weeks = ast.literal_eval(data['parameters']['num_weeks'])
    week_num = ast.literal_eval(data['parameters']['week_num'])
    required_sunday_practices = ast.literal_eval(data['parameters']['required_sunday_practices'])
    required_total_practices = ast.literal_eval(data['parameters']['required_total_practices'])
    data_2023 = ast.literal_eval(data['parameters']['data_2023'])
    # Dates
    dates_string.append(data['dates']['first_tryout'])
    dates_string.append(data['dates']['first_makeup'])
    # Column names
    email = data['columns']['email']
    name = data['columns']['name']
    phone_number = data['columns']['phone_number']
```


```python
def find_date(string):
    date_list = string.split('-')
    month = int(date_list[0])
    day = int(date_list[1])
    year = int(date_list[2])
    return datetime(year, month, day).date()
```


```python
# Convert dates into datetime
dates = []
for i in range(2):
    dates.append(find_date(dates_string[i]))
# New variable names for clarity
week_1_main = dates[0]
week_1_makeup = dates[1]
```


```python
# Date calculation
main_days = [(week_1_main + timedelta(days=i*7)) for i in range(0,5)]
makeup_days = [(week_1_makeup + timedelta(days=i*7)) for i in range(0,5)]
```


```python
# If this fails, the .csv files are in the wrong place
files = glob.glob(r'.\inputs\*.csv')
assert len(files) > 0
if week_num is None:
    week_num = len(files)
    if include_week_0:
        week_num += 1
```


```python
df = pd.concat([pd.read_csv(f) for f in files])
```


```python
# Function to clean email addresses
def clean_address(email):
    try:
        email = email.strip()
        email = email.lower()
        return email
    except:
        return email
```


```python
df[email] = df[email].apply(clean_address)
```


```python
try:
#     tryoutees = df[['Timestamp',email,name,'Name',phone_number]] # for testing
    tryoutees = df[['Timestamp',email,name,phone_number]] # for rollout
    tryoutees = tryoutees.rename(columns={email: 'Email Address', name: 'Full Name', phone_number: 'Phone Number'})
    tryoutees['Email Address'] = tryoutees['Email Address'].apply(clean_address)
except KeyError: # No phone number detected, usually
#     tryoutees = df[['Timestamp',email,name,'Name']] # for testing
    tryoutees = df[['Timestamp',email,name]] # for rollout
    tryoutees = tryoutees.rename(columns={email: 'Email Address', name: 'Full Name'})
    tryoutees['Email Address'] = tryoutees['Email Address'].apply(clean_address)
```


```python
# Merge name columns for 2023 data
if data_2023:
    tryoutees.loc[tryoutees['Full Name'].isnull(), 'Full Name'] = tryoutees.loc[tryoutees['Full Name'].isnull(), 'Name']
    tryoutees = tryoutees.drop(columns=['Name'])
    tryoutees.head()
```


```python
# Assign phone numbers
try:
    list = tryoutees.drop_duplicates(subset='Email Address')
    list = list[['Email Address','Full Name']]
    
    details = df[[email, phone_number]]
    details = details.rename(columns={email: 'Email Address', phone_number: 'Phone Number'})
    details = details.dropna()
    details = details.drop_duplicates(subset='Phone Number')
    details = details.merge(list, how='right', on=['Email Address'])
    details = details[['Email Address','Full Name','Phone Number']]
except KeyError: # Phone numbers missing
    list = tryoutees.drop_duplicates(subset='Email Address')
    list = list[['Email Address','Full Name']]
    
    details = df[[email]]
    details = details.rename(columns={email: 'Email Address'})
    details = details.dropna()
    details = details.merge(list, how='right', on=['Email Address'])
    details = details[['Email Address','Full Name']]
finally: # untested but fixes tryoutees.xlsx
    details.drop_duplicates(subset='Email Address',inplace=True)
    details.reset_index(inplace=True,drop=True)
```


```python

```


```python
# Strip phone numbers
def strip(string):
    string = str(string)
    return re.sub(r'[^0-9]', '', string)
try:
    details['Phone Number'] = details['Phone Number'].apply(strip)
except:
    pass
```


```python
# Print tryoutees list to 'tryoutees.xlsx'
details.to_excel('tryoutees.xlsx')
```


```python
# Convert Google Forms time collection to calendar date
tryoutees['Timestamp'] = pd.to_datetime(tryoutees['Timestamp'], format='mixed')
tryoutees['Timestamp'] = tryoutees['Timestamp'].apply(lambda x: x.date())
```


```python
# Create attendance sheet of all tryoutees
# attendance = tryoutees.drop_duplicates(subset=['Email Address'])
attendance = tryoutees
attendance = attendance['Email Address']
```


```python
# Function checks whether the date is a Sunday or a Makeup day
def check_date(input, main_date, makeup_date):
    if input == main_date:
        return 'Sunday'
    elif input == makeup_date:
        return 'Makeup'
    else:
        return np.nan
```


```python
# Create the columns for the weeks that have passed
tryoutees = tryoutees[['Timestamp','Email Address']]
for week in range(0,week_num):
    # List attendance of each week
    weekly_attendance = tryoutees[(tryoutees['Timestamp'] == main_days[week]) | (tryoutees['Timestamp'] == makeup_days[week])]
    attendance = pd.merge(attendance, weekly_attendance, how='left', on='Email Address')
    attendance.drop_duplicates(subset=['Email Address','Timestamp'],inplace=True)
    attendance = attendance.rename(columns={'Timestamp' : 'Week x'})
    attendance['Week x'] = attendance['Week x'].apply(lambda x: check_date(x, main_days[week], makeup_days[week]))
    attendance = attendance.rename(columns={'Week x' : f'Week {week + 1}'})
    # Flag people who went to both practices
    attendance.loc[attendance.duplicated(subset='Email Address',keep=False),f'Week {week + 1}'] = 'Sunday/Makeup'
    attendance.drop_duplicates(subset=['Email Address'],inplace=True)
```


```python
def tabulate_absences(row):
    count_sunday = sum(value == 'Sunday' for value in row)
    count_makeup = sum(value == 'Makeup' for value in row)
    count_both = sum(value == 'Sunday/Makeup' for value in row)
    count_sunday += count_both  # Going to both days in a week is same as 1 Sunday for attendance
    if count_sunday >= required_total_practices:
        return True
    elif count_sunday == required_sunday_practices and count_makeup >= required_total_practices - required_sunday_practices:
        return True
    else:
        return False
```


```python
if week_num == num_weeks:
    performers = attendance[attendance['Week 5'] == 'Sunday']
    performers = pd.merge(performers, details, how='left', on='Email Address')
    performers['Eligible?'] = performers.apply(tabulate_absences, axis=1)
    new_order = ['Email Address','Full Name','Phone Number','Week 1','Week 2','Week 3','Week 4','Week 5','Eligible?']
    performers = performers[new_order]
    performers.drop_duplicates(subset='Email Address',inplace=True)
else:
    pd.merge(attendance, details, how='outer', on='Email Address')
    attendance = attendance[attendance.iloc[:,-1].notna()]
```


```python
# Prints the people who are eligible and performed Week 5 to 'week_5_performers.xlsx'
if week_num == num_weeks:
    (performers
     .reset_index(drop=True)
     .fillna('N/A')
     .to_excel(f'week_{num_weeks}_performers.xlsx'))
    print(f"Output in week_{num_weeks}_performers.xlsx")
# If before Week 5, print attendance sheet for the most recent week
else:
    (attendance
     .reset_index(drop=True)
     .fillna('N/A')
     .to_excel(f'week_{week + 1}_attendance.xlsx'))
    print(f'Output in week_{week + 1}_attendance.xlsx')
```

    Output in week_5_performers.xlsx
    


```python
atexit.register(input, 'Press Enter to continue...')
```




    <bound method Kernel.raw_input of <ipykernel.ipkernel.IPythonKernel object at 0x000001F6EAE201D0>>


