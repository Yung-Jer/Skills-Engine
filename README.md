
# Skills-Engine

## What is the problem? 

Given a candidate resume and a query of desired role, identify the skill gap between the candidate and the desired role, then generate a unique learning path for the candidate.  

## What is this project for?

This project is a testing playground that I used to record all the progress and how I solve the problems accordingly. 

## Process

### Step 1 Skills Extraction

In order to efficiently extract skills out from the resume, use free online **Emsi Skills** api to get the list of available skills. 

- **Emsi Skills API**

    Example: calling the API endpoint using Python. The detailed documentation is given [Here]([Link text Here](https://link-url-here.org))

    ```
    url = "https://auth.emsicloud.com/connect/token"

    payload = "client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>&grant_type=client_credentials&scope=emsi_open"
    headers = {'Content-Type': 'application/x-www-form-urlencoded'}

    response = requests.request("POST", url, data=payload, headers=headers)
    ```

- **Skills Extraction from Resume**

    Example: extract all the skills from a given resume. The output will be in a Python set.

    ```
    extract_skills(resume_text=resume_string, skills_api=skills_api, clean=True)

    # Output:
    # {
    #     'ada',
    #     'business administration',
    #     'coaching',
    #     'conflict resolution',
    #     'environmental quality',
    #     'labor law',
    #     'management',
    #     'mediation',
    #     'organizational development',
    #     'policy development',
    #     'poultry',
    #     'reduction',
    #     'session',
    #     'strong work ethic',
    #     'teamwork',
    #     'workplace safety'
    # }
    ```

### Step 2 Experience Level Tagging

The output shown above is implicitly indicating that all the skills have the same experience level. This is undesirable as there are much more information in the resume that can be used to weigh the skills differently, e.g. project period. 

In order to weigh each skill differently based on the project period, I first assumed that:
- All the experience levels are using `year(s)` as unit.
- All the skills extracted from the resume have at least 1 year of experience.
- The project description shall come after the project period.
- The year of experience is computed by taking the ceiling of the year difference, i.e. $\text{difference in time} // \text{365 days} + 1$
- When there are multiple projects of different periods having the same skill, the experience level will be computed by $max(\text{years of project 1}, \text{years of project 2}...)$.

Example: extract all the sections out using date range as guidance from a resume.

```
experience_tagging(date_list)

# Output:
# key: year of experience
# value: list of tuples, where each tuple is (section start index, section end index)

# {10: [(981, 1066)], 4: [(1996, 3608)], 3: [(3635, -1)]}
```

## Step 3 Experience Level Tagging

I looped through each section and extract all the skills out, then label all these skills with the corresponding years of experience. For all the other skills that are found not within any section, we just tag the year of experience to be 1. 

Example: extract all the skills with corresponding years from a resume. The output is in a Python
dictionary.

```
skills_experience_level_identification(resume_string)

# Output:
# {
#   'reduction': 4,
#   'environmental quality': 4,
#   'teamwork': 4,
#   'management': 4,
#   'poultry': 4,
#   'strong work ethic': 4,
#   'business administration': 3,
#   'coaching': 3,
#   'conflict resolution': 3,
#   'workplace safety': 3,
#   'mediation': 3,
#   'organizational development': 3,
#   'session': 3,
#   'ada': 3,
#   'labor law': 1,
#   'policy development': 1
# }
```

## Step 4: Identification of Significant Skills

We first assumed that all the candidates that are applying for a specific role **must** be qualified, i.e. they have all the skills required for the particular role. With this assumption, we can do aggregation for all the resumes that belong to the same category and find out which skills appear the most. The skills with high frequency indicate that they are the common skills required for the particular role. 

We then set the significance level $\alpha$ as .9, hence all the skills below 90 percentile will be treated as not so significant. 

Example: We use the **percentile** method to set the threshold for classifying the skills that appear at least once, and decide if they are significant or not. Then, we scale the skills' frequency to the range of [0, 1]. This is to help us to decide whether if we need high level of skill competency to standout from peers. We save all the remaining skill with the corresponding competency level required into a dictionary

```
# agg_data is the dataset after doing groupby(['category']).sum()
# find the threshold of 90 percentile
np.percentile(agg_data['ACCOUNTANT'][agg_data['ACCOUNTANT'] != 0], 90)

# If value > 0.7, skill level required is 5; 
# If 0.7 >= value > 0.3, skill level required is 4;
# If 0.3 >= value, skill level required is 3;
skills_required = {}

for col in agg_data.columns[1:]:
    print(col)
    series = agg_data[col][agg_data[col] > np.percentile(agg_data[col][agg_data[col] != 0], 90)]
    scaled_series = series.apply(lambda x: (x - series.min()) / (series.max() - series.min()))
    binned_series = scaled_series.apply(lambda x: 5 if x > 0.7 else 4 if x > 0.3 else 3)
    skills_required[col] = binned_series.to_dict()
    
# Output: where the value is the competency level based on my client's business settings
# {'ADVOCATE': 
#  {'active listening': 3,
#   'administrative support': 3,
#   'advocacy': 3,
#   'agenda': 3,
#   'auditing': 3,
#   'authorization': 3,
#   'banking': 3,
#   'billing': 3,
#   'budgeting': 3,
#   'business administration': 3,
#   'business development': 3,
#   'c': 3,
#   'case management': 3,
#   'cash register': 3,
#   'collaboration': 3,
#   'community outreach': 3,
#   'conflict resolution': 3,
#   'crisis intervention': 3,
#   'critical thinking': 3,
#   'customer service': 4,
#   'data entry': 3,
#   ...
#  }, 
#  ...
# }
```


