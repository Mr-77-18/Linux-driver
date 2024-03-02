
CHAR DRIVER:
    There are a few kernel apis that could be used to export an interface to userspace in the filesystem(/dev, /sys, etc)

    Devices nodes are one of the interfaces that could be used, where files are exported to users in /dev in the form of char or block device files

    These device files have associated three basic information:
        Type (block or char)
        Major number
        Minor number