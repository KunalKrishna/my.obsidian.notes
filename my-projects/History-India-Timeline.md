
**Main Goal** : To create timeline view of Indian historical events.
**Subsidiary goal** : prepare a large dataset of Indian history (which can help in data analysis)

Vertical Timeline or Scrolling Timeline layout ([Timeline of Hindu Persecution • Hindu Genocide](https://hindugenocide.com/))
##### Key Characteristics of this Design:
1. **Central Spine:** A vertical line that acts as a visual anchor.
2. **Alternating Content:** Placing text on the left and right of the spine (as seen in your screenshot) to balance the page and improve readability.
3. **Milestone Markers:** The use of icons (like the medical cross or slave shackles icons in your view) to categorize different types of events.
### Features

Multiple views based on tags of each events.
#### Basic features : 
- Multiple view to chose from : Can chose one or more
	- War e.g. First Battle of Tarain , Third Battle of Panipat, Kalinga War
	- Religion e.g. Composition of Rig Veda, Fourth Buddhist Council, Bhakti Movement 
	- Administration e.g. Coronation of King Ashoka
	- Economic e.g. Agriculture, trade, taxation, 
####  Advanced features : 
- Additional views 
	- Numismatics :
	- Art
	- Literature
	- Architecture
	- Women
	- Miscellaneous : 
#### Wishful Features : 
- As we scroll timeline from left to right(BCE to AD) we see geographical extension of empires shrink and expand and also empires change( decline of old and rise of new)  with time.
- Based on these dataset create mini-islands of visualization : 
	- Three Round Table Conference 
	- Vacuum created by arrest of Moderates --> Rise of Extremists movements
- Tags on each event based on Exam Importance : UPSC, BPSC, JPSC, SSC, etc. 
	- e.g. Battle of Panipat - `[{UPSC, pre, [2013, 2018]}]` 	- 
	- exam_name
		- stage 1
			- year1
			- year2
		- stage 2
			- year 3
			- year 4
- PRINT Timeline
- PRINT For Practice : 
	- Only time : Write event name
	- Only Event name : write time
	- Zig-zag : some time missing, some event caption missing
- DOWNLOAD TIMELINE
#### Corporate Features : 
- User privilege : 
	- Reader : 
		- consumes(view, download) data 
		- can request new missing date to be added in the database
	- Admin : 
		- approves what gets in the main database (admin dashboard)
		- 
- 

### Challenges : 
Events of two type wrt time : 
1. Point Event : 3rd battle of Panipat (14 January, 1761) |  Battle of the Hydaspes(May 326 BC) 
2. Range Event : Bhakti Movement (7th–17th century) | The Tripartite Struggle (785–816)

Each event's time can be debated : 
1. contested : Rgveda written , 
	1. contested but mostly agreed on 
2. uncontested : Independence of India (15th august 1947)

|                 | point_event                                                     | range_event |
| --------------- | --------------------------------------------------------------- | ----------- |
| is_accurate     | accurate & precise : 3rd battle of Panipat ( _14 January 1761_) |             |
| Is_not_accurate |                                                                 |             |

----------

# Implementation Plan 

## Phase 1 : 

Few attributes, consider simple data type (no date contested)
- caption :
- short description : 
- time : year, month or day
- tags : `[tag1, tag2 ...]`

### Ingestion Pipeline : 




## Phase 2 : 


-----
my prompt


I want to create an educational website on Indian History with following features : 

* ORDERED VIEW : chornological view of historical events using a vertically scrollable timeline (oldest on top) . A vertical straight line in center with time(year) marked . To either side of the marking can be text describing the event. (inspired by : https://hindugenocide.com/ timeline) 
* FILTER: option to select (or deselct) different themes : Politics, Religion, Literature, Art(Painting), Science, War, administration, Famines, ANCIENT, MEDIEVAL, MODERN

Two user group types : 
1. ADMIN: I as owner of the website using admin profile will maintain a "base" form of fact.
2. USER : But I want users to have maximum customization options. User can clone the "base" and customize according to their needs and
* they can customize what to show in date : Year only, full date (DD/MM/YYYY) 
* Citations : book, weblinks, youtube link
* Mulit-format Downaload: Users should also be able to download the factsheet in many forms e.g. csv, excel sheet, pdf, schema etc. 

Each marking on the timeline should have attributes like : 
* CAPTION (short title of event depicted on timeline e.g. First War of Panipat)
* TIME : Year
* Short Description 
* Long Description 
* TAGS : like economics, numismatics, tax, agriculture, administration, book, art, literature, painting, science . These tags will help filter the view. 
* MEDIA : can attach list of images, videos, audios associated with the event 
* CITATIONS: can cite sources of info e.g. books, weblinks etc.

GOAL : Make it fun & interactive for casual user. Plus, memorable and practicable for students esp preparing for competitve exams. 
-----------------
above is my eagle eye view of project. 
now help me : 
1. Add more attributes, features so that in future I wont have problem. 
2. help design history database schema
3. provide overall architecture of the project
4. strategy as to how to populate the schema from textbook(pdf), weblinks ( one idea have thought is to create a chrome extension which i can use while reading webpages to scrap and using AI transform the data in required format into an excel sheet. this sheet can be later used to write to a database. but this approach is very tedius. ).