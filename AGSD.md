[According to crass](https://github.com/rinrin-/crass/blob/master/cui-1.0.4/AGSD/AGSD.cpp), this engine is "Advanced Game Script Decoder" by yamaneko. I have only seen it used in Shinkon Seikatsu by Nicotine Soft. It stores data in a registry key `Software\l-soft\agsd3.0`, suggesting that's v3.0 and the engine is by "l-soft". I do not know of other games using earlier versions.

AGSD uses GSP archives, which are quite simple. The format doesn't support compression or encryption (though of course compressed or encrypted files may be stored in the archive).

AGSD scripts use the SPT file extension, and are a relatively simple compiled script format.