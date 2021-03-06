Not sure what bug made 3 commits come from “wooky” but they should be from me..

# A simple P2P search engine
This is a final project for a data structure class. As of the time I wrote this read me it’s been 1 and half year since. 

Top layer : Tomcat web Servelets: Crawler Servelet, Search Servelet.

2nd layer : Crawler, Searcher

3rd layer : Communication layer

# 1. Crawling
## * Crawler service(DoCrawl.java):
A Tomcat servelet. A GET request can contain a “status” parameter. If the value is “stop” then it tries to stop the crawling process. Otherwise if the status value is barely “status”, or the crawling has already begun, it shows the current status of crawling. 

If no crawling is running, it starts a crawling thread, given 3 parameters:
- seed: The starting point url.
- domain: a general domain url that constrain the crawling process. E.g, if seed is www.bloomberg.com/news, domain can be set to www.bloomberg.com. By this it only crawls webpages that begin with www.bloomberg.com. Or if www.bloomberg.com/news then the crawler won’t crawl other pages in bloomberg.com than those under news/. I didn’t thought of having this until I realized some webpages may link to google thus pretty much the entire internet and I don’t want to crawl it.
- thread number: number of threads to crawl.

This request invokes the Crawler to do the work. The crawler takes another 2 internal parameters that are not given by the user, in addition to the above 3. The 2 parameters are generated by the context in “ChordContext.java”.
- file absolute path: the absolute path to store the crawled files. The way I generated the absolute path for each node is compromised as I had to do demo using our lab’s machines and I did not have full control over them. The problem was that all machines mount the same remote drive(good thing is that I only have to deploy once to the remote drive.). So the only way I can generate a folder for each machine is to name the folder after each machine’s ip address.
- “myself” object of a “Node” type. This is essentially the Communication layer, the lowest layer that takes charge of the data transportation between nodes. Implemented in folder “Chord”. If this object is given as null, then the crawling runs  in standalone mode, meaning no p2p layer and it takes all load and crawls whatever it meets. Otherwise it ignores those pages that are not in its charge.

(Since didn’t have much time working on a class project as this, some static html responses were hardcoded.)

## * Crawler(Crawler.java):
Crawler is of course basically a BFS. 
There’s 1 Crawler instance/thread that spawns any number of parser instances/threads. 
Crawler controls the data structure that stores “remaining links to crawl” and “links already crawled.”.
Each parser object gets links from the “links to crawl” and goes through the links and put new links back to it, as well as the “already crawled.”.
### Serialization:
Every once in a while when 2000 pages are crawled, the crawler stores “links to crawl” and “links crawled” serialized on disk. So it could stop at some point and load later those data when it starts again.

### Caching:
The parsers while parsing pages to get information, it also stores text version of the html file, with its url hashed as file name. So that if internet is down, user can still see the cached text version of the web page when searching.

### Indexing:
Inverted index is used for indexing. For a distributed search engine, as far as I learned, there are basically 2 kinds of design for storing web pages. 
1. Each machine takes charge of a set of keywords and without considering redundancy only it takes charge of those. And it stores all pages that contain any of the keywords. So one page is potentially duplicated among machines because its words are in many machines’ charge.
2. Each machine takes charge of an amount of web pages and the pages won’t be duplicated unless for redundancy purpose. Thus results for 1 keyword scatter among more than 1 machines. This gives more challenge for retrieving search results from machines.

In this case I used the 1st approach, because it’s easier I think, in particular for the p2p implementation where a distributed hash table of keywords is used.

So when a parser does parsing on a web page:
		//process the current page, write it to the file
		//extract all the links from one web page 
		//and put them to the queue
	private boolean parse(String url) {
            Document currentPage;
            //if url is already crawled, return
            if (linkCrawled.contains(url) == true) {
                return false;
            }
            try {
                currentPage = Jsoup.connect(url).timeout(10000).get();
                Elements links = currentPage.select("a[href]");
                for (Element link : links) {
                    String newUrl = link.attr("abs:href");
                    if (newUrl.contains("#") 
				||newUrl.contains("?") 
				|| newUrl.contains("mailto") 
				|| newUrl.contains("jpg") 
				|| newUrl.contains(domainFilter) == false 
				|| linkToCrawl.contains(newUrl) 
				|| linkCrawled.contains(newUrl)) {
                        continue;
                    } else {
                        linkToCrawl.add(newUrl);
                    }
                }
                //Use md5 of the url as the file name
                String fileName = MiscUtil.getHash(url);
                String title = currentPage.title();
                String pageText = getPageText(currentPage);

                storePostingList(fileName, title, url, pageText);
                //write the html for offline caching
                storeOriginalHTML(currentPage, fileName);

            } catch (org.jsoup.UnsupportedMimeTypeException e) {
                return false;
            } catch (org.jsoup.HttpStatusException e) {
                return false;
            } catch (Exception ex) {
                DebugConfig.printAndLog(Crawler.class.getName(), "parse:", ex);
            }

            linkCrawled.add(url);
            return true;
        }

#### The actual indexing is done in “storePosingList(fileName,title,url,pageText)”. (fileName is hashed url)
    //store the posting list into disk
    private void storePostingList(String fileName, String title, 
		String url, String pageText) throws IOException {
        PageInfo currentPageInfo = 
				new PageInfo(fileName, title, url);
        //strip keywords from the text
        String[] keywordsString;
        keywordsString = pageText.split(" ");

		//a unique set of keywords
		//used to get each keyword to process.
		//it’s kind of replicated with keywordsMulti
		//there must be some method to iterate through 
		//unique values of keywordsMulti I probably didn’t 
		//check at that time.
        HashSet<String> keywords = 
		new HashSet<String>(Arrays.asList(keywordsString));

		//used to determine keyword counts for ranking
		//for each keyword we can get its occurrence by 
		//getting its count here
        HashMultiset<String> keywordsMulti = 
		HashMultiset.create(Arrays.asList(keywordsString));
		

        /*
         * for each keyword, create/get its postinglist, 
         * which is a list of all the pages that contain this keyword
         * and then update the postinglist 
         * to include the current page into it.
         * requires synchronized file accessing
         */
        for (String keyword : keywords) {
        	if (stopList.contains(keyword) 
        // _checks if this keyword is current machines responsibility_
        // _for standalone mode, this always returns true_
        // _so the machine always processes all keywords._
			|| takesChargeOf(keyword) == false) {
                continue;
            }

            String keywordHash = MiscUtil.getHash(keyword);
            String urlHash = MiscUtil.getHash(url);
            TreeMultimap<Integer, PageInfo> postingList = null;
            HashSet<String> keywordUrlSet = null;
            ObjectInputStream postingListInStream = null;
            ObjectInputStream keywordUrlSetInStream = null;

            File postingListFile = 
			new File(fileAbsPath + "pageRepo/keywords",
			 keywordHash + "Tree");
            File keywordUrlSetFile = 
			new File(fileAbsPath + "pageRepo/keywords",
			 keywordHash + "Set");

            synchronized (this) {
                //get the postingList from file or create a new one
                try {
                    if (postingListFile.exists()) {

                        postingListInStream = 
			new ObjectInputStream(new FileInputStream(
				postingListFile));
                        keywordUrlSetInStream = 
			new ObjectInputStream(new FileInputStream(
				keywordUrlSetFile));

                        postingList = 
				(TreeMultimap<Integer, PageInfo>)
				 postingListInStream.readObject();
                        keywordUrlSet = 
				(HashSet<String>) 
				keywordUrlSetInStream.readObject();

                    } else {
                        postingList =
				 TreeMultimap.create(
					Ordering.natural().reverse(),
					 Ordering.natural());
                        keywordUrlSet = new HashSet<String>();
                    }
                } catch (Exception ex) {
                    …//catch things
                } finally {
			… //close things
                }
                //put the keyword and page info into the postingList
                if (keywordUrlSet.contains(urlHash)) {
                    continue;
                } else {
                    int keyOccur = keywordsMulti.count(keyword);
                    postingList.put(keyOccur, currentPageInfo);
                    keywordUrlSet.add(urlHash);
                }
                //store the postingList
                ObjectOutputStream listOutStream = null;
                ObjectOutputStream setOutStream = null;
                try {
                    listOutStream = 
				new ObjectOutputStream(
				new FileOutputStream(postingListFile));
                    setOutStream = 
				new ObjectOutputStream(
				new FileOutputStream(keywordUrlSetFile));
                    listOutStream.writeObject(postingList);
                    setOutStream.writeObject(keywordUrlSet);
                } catch (Exception ex) {
                    …//catch things
                } finally {
                    …//close things
                }

            }

        }

    }

Index Structure:

`[
“foo”:
		[10:{title:”a page with foo”,url:”www.a.com/a”,hash:”AHDK2…AWH”},
		 8 :{title:”another page with foo”,url:”www.a.com/b”,hash:”SNJE…3JSF”}],
“else”:
		[…],
…
]`

where foo is keyword, 10 and 8 are the occurrence in each page.
each keyword index is stored as a file, as in `File postingListFile = new File(fileAbsPath + "pageRepo/keywords",keywordHash + "Tree”);`.

Since I had to use our lab’s machines, I didn’t get any database for storing purpose. 

# 2. Searching(Searcher.java)
Search is pretty straightforward, on a machine where a keyword file is existing, it finds the file, the “postingListFile” as said a few lines back. And iterates through it and return a particular portion of it, reflected on the search result page as results for a particular page number.

# 3. P2P implementation(Chord/*.java)
	To be written.