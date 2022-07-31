# dup-finder-node
Duplicate file finder using node - to deduplicate my photo library

I used below code to run on my Cloud folder with 30k+ files to find duplicates.

It searches first based on file-size - grouping all files per file-size

Second part - based on flags - you can remove all files not having the same name

Third part - using content comparison - runs files through a hashing mechanmism to find based on content comparison. This is a very costly function, and can even fail once in a while if you read from a cloud service.

Once all the filtering is done, there is a final run-through to mark files for deletion. This is all very specific to your situation.

Then a number of key checks are needed - 1) not all files of one specific unique match are deleted 2) there are mutliple of one unique match kept.

1. means - tweak filters to be more specific (false positives). 2 means the filters are too specific (false negatives).

The output are multiple things, like error messages as well as DOS-based commands to remove the files. ("DEL"). The master file has the prefix "REM". 

So, this tool requires a lot of manual starting and stopping and checking - but that works for me!

The key element here is the data-structure created in `SameSize`. This is going to be a huge tree with filesizes and the files that are found. And each file entry gets a lots of properties assigned in order to have kill-rules work on a per-file basis. So this also means aggregate properties are stored at file level - a lot of counting and redundant storage done. But that should not be a big issue.

N.B. This code does not do any deletes - it only creates a list of files you could delete.

Placed here on github for future reference.


```
const Path = require("path");
const FS = require("fs");
const crypto = require('crypto');
let Files = [];

let SameSize = {};

// cd "G:\My Drive\Afbeeldingen\archiveren"
// node "G:\My Drive\list.js"

// 
// from  https://stackoverflow.com/questions/18112204/get-all-directories-within-directory-nodejs
function ThroughDirectory(Directory) {
    FS.readdirSync(Directory).forEach(File => {
        const Absolute = Path.join(Directory, File);
        if (FS.statSync(Absolute).isDirectory()) return ThroughDirectory(Absolute);
        else {
            const { size, ctimeMs } = FS.statSync(Directory + '\\' + File);

            if (Array.isArray(SameSize[size])) SameSize[size] = [...SameSize[size], { size, ctimeMs, File, Directory, Path: Directory + '\\' + File }]
            if (SameSize[size] == undefined) SameSize[size] = [{ size, ctimeMs, File, Directory, Path: Directory + '\\' + File }]

            //   if (SameSize[size].length > 1) console.log(SameSize[size].length, '|', File, '|', Directory + '/' + File, FS.statSync(Directory + '\\' + File).size)
            return Files.push(Absolute);
        }
    });
}

ThroughDirectory(".");

// get rid of all filesizes with only 1 entry - they are not double
Object.keys(SameSize).forEach(size => {
    if (SameSize[size].length == 1) delete (SameSize[size]);
});

// console.warn('Same size only double files', SameSize)



// let's find the double names within the same sizes
const sameNameFilter = false;
if (sameNameFilter) {
    Object.keys(SameSize).forEach(size => {
        const fileList = SameSize[size];
        let newFileList = [];

        // let's get all names and calculate the occurences
        const fileNames = fileList.map(file => file.File)
        const nameOccurences = fileNames.reduce(function (acc, curr) {
            return acc[curr] ? ++acc[curr] : acc[curr] = 1, acc
        }, {});

        fileList.forEach(file => {
            const newFile = { ...file };
            newFile['nameOccurences'] = nameOccurences[file.File];
            newFileList.push(newFile);
        });

        // we don't want to see entries which don't have same names existing with same size
        newFileList = newFileList.filter(file => file.nameOccurences > 1);

        if (newFileList.length > 0) SameSize[size] = newFileList;
        if (newFileList.length === 0) delete SameSize[size];
    });

}

// From all double entries - make a hash to find unique content
// https://ilikekillnerds.com/2020/04/how-to-get-the-hash-of-a-file-in-node-js/
const hashFilter = false;
if (hashFilter) {
    const kb = 1024;
    const mb = 1024 * kb;
    const gb = 1024 * mb;
    Object.keys(SameSize).forEach(size => {
        const fileList = SameSize[size];
        const newFileList = [];
        fileList.forEach(file => {
            const newFile = { ...file };
            newFile['hash'] = 'no_hash_size_to_big';
            if (parseInt(size) < 50 * mb) {
                try {
                    const fileBuffer = FS.readFileSync(file.Path);
                    const hashSum = crypto.createHash('sha256');
                    hashSum.update(fileBuffer);
                    const hex = hashSum.digest('hex');
                    newFile['hash'] = hex;
                } catch (e) {
                    console.log('ERRRROR', e);
                }
            }
            newFileList.push(newFile);
        });
        SameSize[size] = newFileList;
    });

    // let's count the unique hashes per file https://stackoverflow.com/questions/5667888/counting-the-occurrences-frequency-of-array-elements
    Object.keys(SameSize).forEach(size => {
        const fileList = SameSize[size];
        const newFileList = [];

        // let's get all hashes and calculate the occurences
        const hashes = fileList.map(file => file.hash)
        const hashOccurences = hashes.reduce(function (acc, curr) {
            return acc[curr] ? ++acc[curr] : acc[curr] = 1, acc
        }, {});

        // let's recreate the filelist and populate the hashCount
        fileList.forEach(file => {
            const newFile = { ...file };
            newFile['hashOccurence'] = hashOccurences[file.hash];
            newFileList.push(newFile);
        });

        SameSize[size] = newFileList;
    });

    // and let's remove the entries with only 1 unique hash - so we are left with files that are pretty sure not uniquely stored
    Object.keys(SameSize).forEach(size => {
        const newList = SameSize[size].filter(item => item.hashOccurence > 1);
        if (newList.length == 0) delete (SameSize[size]);
        if (newList.length > 0) SameSize[size] = newList;
    });
}
// 
// here are the rules to decide which file to delete and which one not
//
Object.keys(SameSize).forEach((size) => {
    const fileList = SameSize[size];
    const newFileList = [];

    // let's add some trigger properties

    // Add property - oldest file
    let oldestFileDate = 9556607963740;
    let oldestCount = 1;
    for (let file of fileList) {
        if (file.ctimeMs == oldestFileDate) oldestCount += 1;
        if (file.ctimeMs < oldestFileDate) oldestFileDate = file.ctimeMs;
    }

    // Add property - flag the ones with "archive" in path, including how many not with this flag
    const archiveList = fileList.map(file => file.Path.includes('archive'));
    const hasItemNotInArchive = archiveList.includes(false);
    const hasItemNotInArchiveCount = archiveList.reduce(function (acc, curr) {
        return curr ? acc : acc += 1;
    }, 0);

    let oldestIndex = -1;
    // let's decide what to keep and what to delete
    fileList.forEach((file, i) => {
        const newFile = { ...file };

        newFile['hasArchive'] = file.Path.includes('archive');
        newFile['isOldest'] = file.ctimeMs == oldestFileDate
        newFile['oldestCount'] = oldestCount;
        newFile['index'] = i;
        newFile['hasItemNotInArchive'] = hasItemNotInArchive;
        newFile['hasItemNotInArchiveCount'] = hasItemNotInArchiveCount;

        if (newFile['isOldest'] && oldestIndex == -1) oldestIndex = i;
        newFile['oldestIndex'] = oldestIndex;
        newFile['hasOldestIndex'] = oldestIndex == i;

        // const killOccurenceInList = fileList.reduce(function (acc, curr) {
        //      return curr.kill ? acc : acc += 1;
        //  }, 0);


        // decide kill or not
        let kill = false;

        // if it has item that is marked as "in archive", kill it first before all others
        if (!kill) {
            if (newFile['hasArchive'] && newFile['hasItemNotInArchive']) kill = true;
            if (kill) newFile['killReason'] = 'is in archive, and we have a file not in archive';
        }

        // if there is only one oldest (others not), kill the younger ones
        if (!kill) {
            if (oldestCount > 1 && oldestFileDate < 9556607963740 && !newFile['isOldest']) kill = true;
            if (kill) newFile['killReason'] = 'we have files that are older';
        }

        if (!kill) {
            kill = newFile['isOldest'] && !newFile['hasOldestIndex'] && newFile['nameOccurences'] > 2
            if (kill) newFile['killReason'] = 'of all oldest this one is not the first in the list';
        }


        if (!kill) {
            kill = newFile['nameOccurences'] > 1 && !newFile['isOldest'] && newFile['nameOccurences'] > 2;
            if (kill) newFile['killReason'] = 'this is not the oldest file';
        }

        // we can still be left with multiple items not with archive in name but
        // if we have multiple items not in archive, kill all but the first found
        //  if (!kill) {
        //   kill = newFile['hasItemNotInArchiveCount'] > 1 && newFile['index'] > 0 && !newFile['hasArchive'];
        //   }

        //   kill = true;
        // assign the kill
        newFile['kill'] = kill;

        newFileList.push(newFile);
    });

    SameSize[size] = newFileList;
});

// safeguard = we cannot kill all files

let filesToKill = 0;
let spacetoFree = 0;

Object.keys(SameSize).forEach(size => {
    const fileList = SameSize[size];
    const hasMoreKeep = fileList.filter(item => !item.kill).length > 1;
    const hasNoKeep = fileList.filter(item => !item.kill).length === 0

    // if there are not issues, let's issue a command
    if (!hasNoKeep && !hasMoreKeep) fileList.forEach((file, i) => {
        const preAmble = file.kill ? 'del ' : 'rem ';
        console.log(`${preAmble} "${file.Path}" `); //"${file.Directory}\\delete_${file.File}"

        if (file.kill) {
            filesToKill += 1;
            spacetoFree += file.size;
        }
    });

    // if (hasMoreKeep || hasNoKeep) console.log(fileList, 'error', hasMoreKeep, hasNoKeep);
    if (hasNoKeep) console.log(fileList, 'error', hasMoreKeep, hasNoKeep);
});


// dump the list to console. So whenever we pipe it to a file, we have the content (e.g. node list.js > doubleEntries.json)
// console.log(SameSize)
console.log('STATS', filesToKill, spacetoFree / 1024 / 1024 / 1024);
```








