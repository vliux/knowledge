# WowTone Android app design spec

## 1.Phrases
- real-time-ASR
- audo-file-ASR

## 2.Design points (which may be ignored)
- one note can relate to 0~N sound records.
- multiple user support.
- user may not logged in yet.
- real-time-ASR may be timeout if 20s no human voice.
- marker points during voice recording.
- OSS file size limit = 128mb.
- keep mappings between each voice slice and each recognized sentence.
- words in a sentence can be marked.
- marked texts should be located seperately, as they can be exported as a stand-alone doc.
- switching to audio-file-ASR if real-time-ASR is not available.
- keeps recording even if screen is off.
- support for fuzzy searching.
- auto scroll the list in voice playing.

---

## 10.Architecture
![wow_tone](https://yuncodeweb.oss-cn-hangzhou.aliyuncs.com/uploads/xhzy-android/wow-tone-app/c9bc2f7213774f9608f369e001f76c36/wow_tone.jpg)

## 20.Model design
Data of difference users should be isolated by difference DBs.
Example: data of "vliux" should be in DB_vliux.

### 20.1.Table "Note"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| id   | int   | N | primary key |
| title   | text   | N | |
| ctime   | date   | N | creation time |
| mtime   | date   | N | last modified time |
| deleted   | bool   | Y | is deleted? |
| abs | text | Y | abstract |

### 20.2.Table "Voice"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| id   | int   | N | primary key |
| nid   | int   | N | note id (FK to Note) |
| stime   | date   | N | time when the voice is started recording |
| dur | long   | N | voice duration time |
| url | text | N | server side url of the voice file |
| file | text | Y | client side url of the voice file. Null means the file is not yet downloaded. |
| deleted   | bool | Y | is deleted? |

### 20.3.Table "VoiceMark"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| id   | int   | N | primary key |
| vid   | int   | N | voice id (FK to Voice) |
| ctime   | date | N | creation time |
| offset   | long   | N | time offset of this mark inside voice |
| desc | text | Y | |

### 20.4.Table "Sentence"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| id   | int   | N | primary key |
| nid | int | N | note id (FK to Note) |
| vid   | int   | Y | voice id (FK to Voice), null if it is added by user manually |
| ctime   | date | N | creation time |
| offset   | long   | Y | time offset of the start of sentence inside voice(ms), null if it is added by user manually  |
| dur | long | Y | duration of the sentence(ms), null if it is added by user manually |
| text | text | N | |
| deleted | bool | Y | |

### 20.5.Table "SentenceMark"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| id   | int   | N | primary key |
| nid | int | N | FK to Note |
| sid | int | N | FK to Sentence |
| start | int | N | position offset of start |
| end | int | N | | position offset of end |

### 20.6.Table "Tag"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| id   | int   | N | primary key |
| name | text | N | |

### 20.7.Table "NoteTag"
| Field | SQL Type | Nullable | Desc |
| -------- | -------- | -------- | -------- |
| nid   | int   | N | note id |
| tid | int | N | tag id |

'nid'+'tid' should be unique.

### 30.RAM data structure
For Note whose voice is translated, the content can be rendered as a ListView/RecyclerView.
```
class NoteContentListItem {
  long startOffset, endOffset; // time offset mapping to voice
  @NonNull Note n;
  @Nullable Sentence s;
  @Nullable VoiceMark[] vms;
  @Nullable SentenceMark[] ss;
}
```
Consider caching the items in memory after exiting from Note content page.
---

## 40.Note management
### 40.1.Note creation
#### 40.1.1.Real-time-ASR
Use iDST real-time-ASR SDK to obtain the sentences of the voice, while at the same time progressively save the PCM data to local disk. The PCM data should then be converted to .WAV, or if possible then to .MP3 for less space occupation. 

#### 40.1.2.Audio-file-ASR
Use AudioRecoder or MediaRecord to save the voice to a .WAV or .MP3 file locally. Enable the NetStatReceiver. When wifi is connected, start a Service to upload the Note with audio files, obtaining translated sentences from Zhiyun backend server (instead of interacting with iDST server).

**Show a notification when the Service is working, to reduce the possibility of process killing.**

**Disable NetStatReceiver after all works have been done.**

### 40.2.Note editing
For translated note, existing sentence is editable. New sentence can be inserted into the middle/start/end of all sentences.

### 40.3.Note abstract
Note abstract (Note.abs) is used as cache for rendering note list. Upon each note saving, this field should be updated.

### 40.4.Note synchronization
When app is at foreground, sync every X seconds. 
When app is at background, sync every Y hours:
- on Android 5.0+, use JobScheduler with prerequisites: connection, charging stat.
- on Android 5.0-, fall back to NetStatReceiver+AlarmManager. Note the receiver component should be disabled on Android 5.0+. 

If a note is in below conditions during sync, re-sync that note after upload/file-ASR:
- a Note and any of its res is being uploaded.
- a Note is in the process of audio-file-ASR.
That means a note-level lock should be enforced in sync/upload/file-ASR.

### 40.5.Conflict handling
Server side should detect conflicts during sync or submit, and notify the client. On receiving conflicts, app should rename the current note to a new name like "original_name_冲突" and thus in effect creating a new note.
The new note clones all the data from the original one. **But duplicated downloading of the same voice file should be avoided by mechanisms like HTTP caching**.

### 40.6.Note searching

---

## 50.User management
### Registration

### Log-in

### Log-off

### Change password

### QR scan for web log-in

## 60.Version upgrade

