# Firefox JSON Password Parser

This python script can convert the password JSON from firefox to CSV, containing URL, Username and Password.
Additionally, it separates the URL into 2 parts (`https://www.` and `example.com`) and alphabetically sorts them.

# Howto

First, you need to obtain the passwords from firefox. You can use the following javascript code from [here](https://support.mozilla.org/de/questions/1077630#answer-834769) in the JavaScript console of Firefox after setting `devtools.chrome.enabled` to `true` in `about:config` (Credit to  cor-el from support.mozilla.org).

    /* export the names and passwords in JSON format to firefox-logins.json */
    var tokendb = Cc["@mozilla.org/security/pk11tokendb;1"].createInstance(Ci.nsIPK11TokenDB);
    var token = tokendb.getInternalKeyToken();
    
    try {token.login(true)} catch(e) {Cu.reportError(e)}
    
    if (!token.needsLogin() || token.isLoggedIn()) {
     var passwordmanager = Cc["@mozilla.org/login-manager;1"] .getService(Ci.nsILoginManager);
     var signons = passwordmanager.getAllLogins({});
     var json = JSON.stringify(signons, null, 1);
    
     var ps = Services.prompt;
     var txt = 'Logins: ' + signons.length;
     var obj = new Object; obj.value = json;
    
     if (ps.prompt(null, 'Logins - JSON', txt, obj, null, {})){
     var fp=Cc["@mozilla.org/filepicker;1"].createInstance(Ci.nsIFilePicker);
     fp.init(window,"",Ci.nsIFilePicker.modeSave);
     fp.defaultString = "firefox-logins.json";
    
     fp.open((rv) => {
     if (rv == Ci.nsIFilePicker.returnOK || rv == Ci.nsIFilePicker.returnReplace) {
     var fos = Cc['@mozilla.org/network/file-output-stream;1'].createInstance(Ci.nsIFileOutputStream);
     fos.init(fp.file, 0x02 | 0x08 | 0x20, 0666, 0);
     var converter = Cc['@mozilla.org/intl/converter-output-stream;1'].createInstance(Ci.nsIConverterOutputStream);
    
     converter.init(fos, 'UTF-8', 0, 0);
     converter.writeString(json);
     converter.close();
    }})
    }}

Just store the JSON output in some file, e.g. `pws.json`. Then download the script and pipe the passwords into it. You can also store the output in a file, e.g. with `cat pws.json | python3 fparser.py > pws.csv`. If you want to copy the script without downloading, here you go:

    #!/usr/bin/env python
    import sys
    import json
    import numpy as np
    
    json=json.load(sys.stdin)
    uList = []
    pwList = []
    urlList = []
    for a in json:
    	# Split into domain, tld, etc.
    	line = a['hostname'].strip().split('.')
    	recent = [e+'.' for e in line if e and line.index(e) != len(line)-1]
    	recent.append(line[-1])
    
    	# Handle cases that are not something.domain.tld
    	# Sadly doesn't handly ...co.uk etc., have to do manually
    	if len(recent) != 3:
    
    		# Assume stuff starts with something://, fail assertion otherwise
    		recent[0] = recent[0].split('//')
    		assert(len(recent[0])==2)
    		recent[0][0] = recent[0][0]+'//'
    
    		# Deal with different cases
    		if np.shape(recent) == (1,2): #http://localhost:631
    			recent = [recent[0][0],recent[0][1],' ']
    		if np.shape(recent) == (2,): #https://example.com
    			recent = [recent[0][0],recent[0][1],recent[1]]
    		if len(recent) >= 4: #https://www.login.service.example.com
    			recent = [recent[0][0]+recent[0][1]+''.join(recent[1:-2]),recent[-2],recent[-1]]
    	# urlList contains array [stuff, domain, tld], uList and pwList just usernames and pws respectively		
    	urlList.append(recent)
    	uList.append(a['username'])
    	pwList.append(a['password'])
    
    # Sort by domain
    urlList = np.array(urlList)
    sortList = []
    for a in urlList:
    	sortList.append(a[1])
    sortList = np.array(sortList)
    indices = np.argsort(sortList)
    uList = np.array(uList)
    pwList = np.array(pwList)
    
    # Print as csv with delimiter | since , often in pws
    for i in indices:
    	output = urlList[i][0]+'|'+urlList[i][1]+urlList[i][2]+'|'+uList[i]+'|'+pwList[i]
    	print(output.encode('utf-8'))

If you have any questions or remark, feel free to contact me here.

# History
I regularly print out the stored passwords from Firefox for some family members. Since the addon we used before doesn't work anymore in Firefox Quantum, I decided not to use the next addon but to write something myself. The benefit is that you don't have to trust the author of some addon and you can easily check the script (ca. 50 lines) youself.