# Chapter 3: The Variables


There are two kinds of people in the world: those who already know what variables they need and those who have not yet identified them. Part One is for those who already have their variables and Part Two is for those searching for their variables.

Part One: When You Know Your Variables

I cannot provide the data — it is not mine to share — but I can provide the variables to pull the data to run this analysis (In Part Two I will show you how to search for other variables.)

Identification variables:
- 
​
ER30001, ER30002 — used together to build person_id
- 
​
ER30004 — age in 1968 (to derive birth_year)
- 
​
ER32006 — sample status (filter to valid respondents)

​

Parent (1968) Variables:
- 
​
V103 — parent homeownership
- 
​
V313 — parent education
- 
​
V81 — parent income 1968
- 
​
V181 — race
- 
​
V93 — state 1968
- 
​
V119 — sex of 1968 head

​

Child Outcome Variables:

ER32000 — child sex

ER35152 — child education years

ER82032 — child homeownership 2023



Go to the PSID webiste

![]()

Once registered click on Data. Then Data Center. But notice FIMS below. You will need that later.

![]()



Clicking on Data Center will bring this screen up

![]()

Click on Variable List. 

![]()

Here is the list of every variable you need to recreate this project.  Copy & Paste them into the Variable List box, either one per row or with commas between them.

ER30001
ER30002
ER32006
V103
V313
V81
V181
V93
V119
ER32000
ER35152
ER82032

Clicking on Add Variables moves them to your cart

![]()

That brings this up

![]()

I guess these are to refine your variable selection but we will ignore these and select instead Checkout. The codebook is the Rosetta Stone. Leave Codebook as a pdf. Select Microsoft Excel Spreadsheet. Leave the rest of choices as shown here.

![]()

In a moment you will have your data. Your xlsx file will have a different name. I prefer to download each of the 3 files separately. If you select the download all button you will get a zip file. In any event, download them and move them into a dedicated folder.

Open the xlsx file in Libre Calc or Excel and save it as a csv file. ( Note it probably won't open in Google sheets and maybe some other programs because it is too big.)

The text file looks just like you imagine it would but the Codebook.pdf is interesting. Everything that you could want to know about your variables is contained here. Frankly just upload it to your favorite AI and query that instead of flipping back and forth through the pages.

![]()

Part Two: But What If You Were Born Not Knowing Which Variables You Needed?

For clarity we will only search for variables related to Education. But for those regarding home ownership would be done the same way.

![]()

Then the Data Center

![]()

Now things get murky for me. Real murky. We have to choose between Individual Data Index and the Family Public Data Index. The Individual level data are things like Age, Education, etc. The Family level reports on household things like housing costs, food expenses, location, family size. But in reality I have a feeling the two are are not so neatly separated. But then, I might be dead wrong on this.  As a rule, I grab any data that seems even vaguely related.

Inconveniently, not all variables are available in each survey wave. The Cross Year Index lets one see what years your variable is available.

Select the Cross-Year index

![]()

Notice at the top of the list the choice between the Individual Data Index and the Family Public Data Index

Let us expand the individual and Family Indexes and look for Educational attainment in each.

First the Individual Index.

![]()

And this from the Public Family Index

![]()

Next, click on Add to Cart

![]()

We saw this earlier. Make these selections and hit Submit.

![]()



In a few minutes an email will arrive. They have your email because only registered users can pull data. The email contains a link to the data. The data is in xlsx format. Change it to csv format. It has too many rows for Google Sheets so I download it and then I open it from my downloads folder into LibreOffice Calc then save it as a csv. The file names are not attached. That may be for the best because I think the names would be so very similar that they would be confusing in themselves. Later, after we winnow down the variables, we will apply the labels. The codebook is our Rosetta Stone. Just feed it into some AI like Claude.

![]()

But before you can use any of those variables, there is one more piece of the puzzle.

Part 3 of our 2 part chapter: FIMS (with apologies to Click & Clack)

FIMS stands for Family Information Management System and it is the sinews that hold all of the bones together to make the corpus. Download the FIMS User Guide. In addition to the PSID Codebook I would upload this into your favorite AI tool as well.

Back to the Home screen: Select FIMS.

![]()

Select Inter-generational. This starts with the grandkids and goes back in time. Wide format so each row has the characteristics of single person. And excel format. Notice FIMS does not ask about Individual or Family indexed data.

After you hit submit a link labeled Excel File will appear below the Submit button. If you do not see it right away , perhaps scroll down a little. Clicking the link will download the file.

![]()

And this is what FIMS data looks like:

![]()







Next up: turning these variables into a working dataset — and why the first notebook is the hardest one you'll never have to write again















 












