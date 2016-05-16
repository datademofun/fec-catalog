# Exploring the FEC Expenditures


The Independent Expenditure and Candidate Disbursements data can be found at at these two URLs:

- [http://www.fec.gov/data/IndependentExpenditure.do](http://www.fec.gov/data/IndependentExpenditure.do)
- [http://www.fec.gov/data/CandidateDisbursement.do](http://www.fec.gov/data/CandidateDisbursement.do)


# Fun with Python

Let's first explore the [Independent Expenditures data](http://www.fec.gov/data/CandidateDisbursement.do). There's a clickable icon for getting the data in CSV format; I've hardcoded it below for 2016:

~~~py
from datetime import datetime
from collections import defaultdict
import csv
import requests
url = 'http://www.fec.gov/data/IndependentExpenditure.do?format=csv&election_yr=2016'
~~~


The data doesn't come in perfect form; notably, we have to clean up the money amounts and date values. So I like creating two little helper functions:

~~~py
# converts "$123,233.44" to 123233.44 (a float)
def clean_money(xstr):
    xs = xstr.replace('$', '').replace(',', '')
    return float(xs)

# converts '11/04/2016' to date object
def clean_date(xstr):
    return datetime.strptime(xstr, '%m/%d/%Y')
~~~


## Download and de-serialize

Download the data and de-serialize it:

~~~py
resp = requests.get(url)
rows = list(csv.DictReader(resp.text.splitlines()))
~~~


The data fields will sometimes have stray spaces, so, for each row (and remember that each row is a __dictionary__), we reassign the value to be the string _stripped_ of trailing whitespaces, i.e. `"123 "` is converted to `"123"`:

# Clean the data strings

~~~py
for row in rows:
    # strip out trailing whitespaces
    for k in row.keys():
        row[k] = row[k].strip()
~~~

And then a second loop to fix up the date and dollar values:


## Typecast the values

~~~py
for row in rows:
    row['agg_amo'] = clean_money(row['agg_amo'])
    row['exp_amo'] = clean_money(row['exp_amo'])
    if row['rec_dat']:
        row['rec_dat'] = clean_date(row['rec_dat'])
    if row['dissem_dt']:
        row['dissem_dt'] = clean_date(row['dissem_dt'])
~~~


Of course, you could always just use one main loop and do all the cleaning in one go:

~~~py
for row in rows:
    # strip out trailing whitespaces
    for k in row.keys():
        row[k] = row[k].strip()
    row['agg_amo'] = clean_money(row['agg_amo'])
    row['exp_amo'] = clean_money(row['exp_amo'])
    if row['rec_dat']:
        row['rec_dat'] = clean_date(row['rec_dat'])
    if row['dissem_dt']:
        row['dissem_dt'] = clean_date(row['dissem_dt'])
~~~



Some example queries:


### To get the total spent on independent expenditures

~~~py
sum([r['exp_amo'] for r in rows])
~~~

### To find how much money was spent _against_ Trump:

~~~py
x = 0
for r in rows:
    if 'trump' in r['can_nam'].lower() and 'Oppose' == r['sup_opp']:
        x += r['exp_amo']  
~~~


### The  spenders _against_ Trump:

For each spendername, track the spending:

~~~py
spenders = defaultdict(int)
for r in rows:
    if 'trump' in r['can_nam'].lower() and 'Oppose' == r['sup_opp']:
        spenders[r['spe_nam']] += r['exp_amo']
~~~


### Get spending by Senate state race


~~~py
# total by senate state
senate_totals = defaultdict(int)
for row in rows:
    if row['can_off'] == 'S':
        st_name = row['can_off_sta']
        senate_totals[st_name] += int(row['exp_amo'])


for name, total in sorted(senate_totals.items(), key=lambda o: o[1]):
    print(str(total).rjust(12), name)
~~~




# Get top Senate races by candidate disbursements


Here's a start (downloading from FTP requires a different library, but it's the same process):

~~~py
from datetime import datetime
from collections import defaultdict
from urllib.request import urlopen
import csv
url = 'ftp://ftp.fec.gov/FEC/data.fec.gov/candidate_disbursement2016/all_senate.csv'

# download and deserialize the data
with urlopen(url, 'r') as u:
    lines = u.read().decode('utf-8').splitlines()

rows = list(csv.DictReader(lines))
~~~
