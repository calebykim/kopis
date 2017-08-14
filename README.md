# Kopis
A CLI tool to search and render the metadata of Marathon-deployed Mesos applications.  

## Description
Kopis can be used to:
1. Take resource inventory of Marathon orchestrated Mesos clusters
2. Identify differences in Marathon application configurations 

Kopis provides this data by supporting custom user queries via command line. 

Essentially, users can ask Kopis for the memory, cpus, disk, or any attribute of all applications with a certain id-substring, a label, environment variable, etc. This CLI is a flexible tool that can search for any key(s), value(s), or key-value pair(s) of the Marathon application JSON objects and render the results in useful ways. 

Some use cases:
1. Sum the resource usage of a certain service across all environment. 
2. Find all applications that are in enviornment '/mobile' and are using .1 cpus, print the results in CSV/JSON format.
3. Return the healthCheck protocol of app with id: '/myapp'.

Kopis is versatile, and with some Bash it is an even more powerful CLI that can used to effectively sift through and extract Marathon config data to better manage Mesos clusters. 

### Kopis Implementation and Flow
When Kopis is first run, the program pulls all the Marathon app data from Marathons '/v2/apps' endpoint. It then caches the data in a local JSON file for a configured TTL for efficiency, and runs its search through the local file. 

Kopis utilizes a deep recursive search algorithm to find key/value(s) in the mutli-nested json object pulled from Marathon's REST api. Upon finding the target item, Kopis prints the results in STDOUT in the user's requested format. 
        
## Installation:
### Requirement: **python3**

1) Install Kopis:
       To be released.
        
2) Configure Marathon instance (required) and other optional settings.
        
       run: 'kopis touch' to create ~/kopis directory and yaml config file in ~/kopis directory 
            In ~/kopis/kopis.yml:
                MARATHON_URL : '' # set this to your Marathon instance url (required)
                MARATHON_USER_NAME : '' # set basic auth username for Marathon (optional)
                MARATHON_PASSWORD : '' # set basic auth password for Marathon (optional) 
                CACHE_TIMEOUT : 5  # every x minutes Kopis will refresh the Marathon data (optional)
                CSV_DELIMITER : ',' # set CSV delimiter (optional)

   Set the MARATHON_URL. It must end with the domain name or port, e.g., http://marathon-int-example.com or http://marathon-int-example.com:9999'. Ensure that there is no '/' at the end of url.
   
   The CACHE_TIMEOUT is the minute interval Kopis will automatically reload its cached Marathon data.
   
 ##### Installation done!
    
## User Guide
 #### Example query:
 
        kopis search 'id=*app* and (mem=1024 or mem=2048)'
 This query finds all applications that have 'app' in their ids and use either 1024 or 2048 MiB. 
 
 #### Command line Syntax:
    The kopis command is 3 parts:
    
    1) 'kopis' - to run the executable
    2) 'search' - the search command
    3) 'query' - the conditions to search for 
   
        kopis search '<query>'
        
    optional arguments:
        -r, -refresh  refresh application data from Marathon 
        -m, -marathon override marathon url in config with given url
        -v, -verbose  verbose option and pretty print
        -h, --help    show this help message and exit
       
        Data Rendering Options:
        -s, -sum     prints sum of provided -json or -csv keys, default prints sum of cpus, disk, mem 
        -j, -json    builds custom json with given keys ex: '-j cpus mem disk'
        -c, -csv     builds custom csv  with given keys ex: '-c cpus mem disk'
        -u, -url     print urls of results
        -f, -full    print full json object of results
       

        
#### Query Syntax

  + Surround entire query with single quotes and any tokens you don't want split with double quotes.
                
        kopis search 'key="value with spaces"' # "value with spaces" will be evaluated as one token
  
  + Key-value pairs must have an operator sign between them and no whitespace
  
        correct: 'key=val'  
        incorrect: 'key = val'
  + When using ! operator, place it in front of the value. For key-val pairs use '!='
  
        correct: '!port', 'key!=value'
        incorrect: 'port!', !key=value'
        

## Features

### Search Features

  #### A query can consist of a key, a value, or a key value pair.
   
    kopis search 'docker' # searches for 'docker' as a key or a value
    kopis search 'cpus=.1' # searches for key:'cpus' that has value .1
  
  #### Wildcard Id matching can be used to find applications with matching ID substrings using '*'
  
    kopis search 'id=/dev/*/service' # finds ids that start with '/dev/' and end with '/service'
    
  #### You can join queries using 'and' or 'or' operators with parentheses as well.
  
    kopis search 'docker and (cpus=.1 or mem=512)' # searches for apps with docker and that have either .1 cpu or 512 mem
    
  #### Search for specific nested keys using dot notation. 
  
    kopis search 'healthChecks.protocol=HTTP' # searches nested 'protocol' key in 'healthChecks' object
   
  #### Comparator Operators: '<', '>', '<=', '>=', '!=', '=', and '!' are supported.
    
    kopis search '(cpus>.1 or mem<=1024) and !docker ' # searches for applications that have cpus greater than .1, mem less than or equal to 1024, and that do not contain 'docker' in any field
    
 
 ### Data Rendering Features
 
   #### -s, -sum option returns the total resource usage of result or sum of json, csv keys
    Input: kopis search 'id=*dev*' -s -j cpus mem disk
    Output:
      {
        "cpus": 23.0,
        "mem": 23340,
        "disk": 12500
       }
    *prints sum cpus, mem, and disk of matching applications 
    
   #### -j, -json option outputs matching applications in json with given keys (space seperated)
   
    Input: kopis search 'id=*dev*' -j cpus mem disk -v 
    Output:
        [
            { 
                id: dev/app1
                cpus: .1
                mem: 2048
                disk: 500
            },
            
            { 
                id: dev/app2
                cpus: .2
                mem: 1024
                disk: 500
            }, ...
        ]
    * without -v argument, result will print in one line    
   #### -c, -csv option outputs matching applications in csv format with given keys (space seperated)
   
    Input: kopis search 'id=*dev*' -c cpus mem disk
    
    Output: 
        id,mem,cpus,disk
        dev/app1,.1,2048,500
        dev/app2,.2,1024,500
        ...
        
     *You can redirect the output into .csv file using bash
     
  #### -u, -url option outputs urls of the applications' Marathon configuration pages
  
    Input: kopis search 'id=*dev*' -u
    
    Output
        http://marathon-example/ui/#/apps/%2Fdev%2Fapp1/configuration
        http://marathon-example/ui/#/apps/%2Fdev%2Fapp2/configuration
        ...
        
     *Click urls from terminal with cmd/ctrl click

  #### -f, -full option outputs entire applications' configuration jsons
  
    Input: kopis search 'id=*dev*' -f -v
    Output:
      [
        {
          "id": "/myapp",
          "cmd": "env && sleep 60",
          "args": null,
          "user": null,
          "env": {
            "LD_LIBRARY_PATH": "/usr/local/lib/myLib"
          },...
        }...
      ]
    * without -v argument, result will print in one line    

        

  
        
        
        
    
     
    
    
 
 



