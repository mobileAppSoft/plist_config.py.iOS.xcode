# plist_config  

###### Mon Nov 11 14:59:33 MSK 2019 started  

## Problem
> I have an iOS app with multiple targets...
> Can plists inherit from a common parent plist? I would like to reduce duplication in plist files...  
https://stackoverflow.com/questions/24789464/ios-plist-inheritance

## Solution
Make plist generator (`plc` cli utility) which enables to compose plist from reusable parts of plist.  
    So, a work the utility should do is:  
    \0. Scan existing `.xcodeproj`, see what targets are there and generate `plist_config.plc` in accordance with those.
    \1. Parse plist_config_file (user specifies the config following the syntax we require)  
    \2. Generate plist for each target specified in the config  
    \3. Modify XCode build config so that all specified targets point to generated plists  

---

## Spec  
The spec should provide the extensive overview of utility behavior and certain implementation details. Each point of the spec has numeric id so it should be easy to point to the problems one by one.  
    
\#10 is Implementation Plan. Refer to it to go on with the work.  

MAX id: 24  

---

\4. CLI command name:  
    `plc` – stays for "PList Config"

\15. `.plc` – plist_config file extension  

\19. plc CLI available commands:  
    * init  
    * install  
    * version  
    * help  

\5. Plist config_file example
    ```
    
    #target ProjectTarget1
        #include common plist part

    #target ProjectTarget2
        #include common plist part  
        #include light theme only  
        #include 
            <key>CFBundleDevelopmentRegion</key>
            <string>en</string>
            <key>CFBundleDisplayName</key>
            <string>NATIVE TALK</string>
            <key>CFBundleExecutable</key>
            <string>${EXECUTABLE_NAME}</string>
            <key>CFBundleIcons</key>
            <dict/>

    #target ProjectTarget3
        #include ProjectTarget3-Info.plist

    // sample comment
    #def common plist part
        #include
            <key>UIStatusBarHidden</key>
            <true/>

    #def light theme only
        #include
            <key>UIUserInterfaceStyle</key>
            <string>Light</string>
    ```

    So, the full list of keywords is: #target, #def, #include, #header, #footer
    `#` is a part of the keyword and is used as a keyword_marker that can be seen easily while using any text editor regardless of having any highlighting features within the editor.  

\16. plc file content syntax:  
    ```  

    // this is example of a comment  

    // `(_ : 1)` means `_` have to be present exactly one time  
    // `(_ : 1..*)` means `_` have to be present at least one time, but not limited to one   
    // `(_ : 0..*)` means `_` can be present any number of times  
    // `<_>` is a placeholder and means `_` is a user-specific content  
    
    ((#target <target name> : 1)
        (#include (<link_to_#def_block> : 1) : 0..*) // fail if cycling detected
        (#include (<relative_path_to_plist_file> : 1) : 0..*)
        (#include (<row_plist_part> : 1) : 0..1) // validate row_plist_part
    : 1..*)

    // all #def blocks should be placed below all #target blocks
    // name of the #def can contain spaces
    ((#def <some name> : 1)
        (#include (<link_to_#def_block> : 1) : 0..*) // fail if cycling detected
        (#include (<relative_path_to_plist_file> : 1) : 0..*)
        (#include (<row_plist_part> : 1) : 0..1) // validate row_plist_part
    : 0..*)

    ((#header : 0..1)
        (#include (<link_to_#def_block> : 1) : 0..*) // fail if cycling detected
        (#include (<relative_path_to_plist_file> : 1) : 0..*)
        (#include (<row_plist_part> : 1) : 0..1) // validate row_plist_part
    : 0..1)

    ((#footer : 0..1)
        (#include (<link_to_#def_block> : 1) : 0..*) // fail if cycling detected
        (#include (<relative_path_to_plist_file> : 1) : 0..*)
        (#include (<row_plist_part> : 1) : 0..1) // validate row_plist_part
    : 0..1)
    ```  

\23. Validate order of #include of different types within one #def, or #target, or #footer, or #header block.  
    I. e. consider the following order should be considered as invalid:  
        #include <row_plist_part>
        #include <link_to_#def_block>  

\21. Validate <row_plist_part> as valid plist content  

\22. Protect from cycle references while resolving #def _ #include _

\18. Other plc parsing rules:  
    * indentation agnostic
    * line breaks agnostic
    * it's basically should be tokenized into the list of pairs like {keyword content} where content  
        * of #include, #header, #footer can be empty  
        * of #def, #target has to be not-empty  

    * taking into account all of the points above, spaces between  
        * keyword and the following content,  
        * and content and the next keyword  
        should be ignored  
        I. e. the whole plc can be minified to one line.  

Conflicts resolution policy:  
    \6. any row_plist_part #include should override any of the same properties included using link_to_#def_block or relative_path_to_plist_file `#include`  

    \7. conflicts between several link_to_#def_block or relative_path_to_plist_file `#include` within the same target shouldn't be resolved automatically.  


\8. generate plist for each #target block

\9. Modify build config of targets (only those that are specified in plc) so that all specified targets point to generated plists  

\10. Implementation plan  

    \11. Install python `v.3.7.6`. Currently it should be the default version you get when running `brew install python`.  

    Then I would start with the following:  
    \12. implement the script (cli app) that  

        * can be run as `<absolute_path_to_script>/plc init`

        * should be run while `pwd` == directory with at least one `.xcodeproj`  
            * report error if no `.xcodeproj` found  
            * report error if more than one `.xcodeproj` found in the same dir  
        
        * creates `plist_config.plc` file with the following content:  
            ```  
            
            #target <target1>
                #include original <target1> plist

            #target <target2>
                #include original <target2> plist 
                

            #def original <target1> plist 
                #include <relative path to plist that has been already specified for target1 in existing .xcodeproj>
            
            #def original <target2> plist 
                #include <relative path to plist that has been already specified for target2 in existing .xcodeproj>

            // and so on for all the targets that already exist in .xcodeproj  
            ```  

\13. On each run of `plc init` generate default (if it does not exist already) `.plcconfig` in <user_home_dir> with the following content  
    ```  

    #header  
        #include  
            <?xml version="1.0" encoding="UTF-8"?>
            <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
            <plist version="1.0">
            <dict>

    #footer
        #include 
            </dict>
            </plist>
    ```  

    \14. These #footer and #header should be included into each generated `.plist` file  

\20. During `plc init` automate the process of finding duplicated parts of existing plists and generate appropriate #def blocks.  



---
---

## Done items  
Move all completed items to the bottom of this file like this  

[x] ###### Fri Jan 3 17:15:33 MSK 2020  
    \24. Publish initial spec to github  

