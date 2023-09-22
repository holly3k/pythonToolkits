from __future__ import annotations
import os
import json
from tkinter import *
from tkinter.ttk import *
from tkinter import filedialog
from collections import namedtuple
import ctypes

class FileTypeAndSize:
    size = 0
    readableSize = ""
    type = ""
    path = ""
    childs:list[FileTypeAndSize]

def customDecoder(geekDict):
    return namedtuple('X', geekDict.keys())(*geekDict.values())

signalToStop = False
def sortKey(typeAndSize:FileTypeAndSize):
    return typeAndSize.size

def getFileName(path):
    index = path.rfind('\\')
    if (index > -1) and (index != len(path) -1) :
        return path[index+1:]
    else:
        return path
    
def insertFolderNode(tree:Treeview, parent, typeAndSize:FileTypeAndSize):
    sf1 = tree.insert(parent, END, text=getFileName(typeAndSize.path),open=False,values=(typeAndSize.readableSize,typeAndSize.path))
    if len(typeAndSize.childs) > 0:
        typeAndSize.childs.sort(key=sortKey,reverse=True)
    for child in typeAndSize.childs:
        if child.type == 'FILE':
            tree.insert(sf1, END, text=getFileName(child.path),open=False,values=(child.readableSize,typeAndSize.path))
        else:
            insertFolderNode(tree,sf1,child)

def insertFolderNodeFunc(tree1):
    global currentFolderAndSize
    tree1.delete(*tree1.get_children())
    insertFolderNode(tree1,'',currentFolderAndSize)
    print('finished insert node')
    statusvar.set('finished')



def recursive_listdirThread(tree1,rootPath,):
    global signalToStop,currentFolderAndSize,p
    signalToStop = False
    lib = ctypes.cdll.LoadLibrary("./folderSizeWalker.dll")
    startScanning = lib.startScanning
    startScanning.argtypes = [ctypes.c_char_p]
    startScanning.restype = None
    startScanning(rootPath.encode())
    statusvar.set('Scanning....')
    statusbar.update()
    f = open("./out.json", 'r', encoding='utf-8')
    r = f.read()
    currentFolderAndSize = json.loads(r, object_hook=customDecoder)
    signalToStop = True
    statusvar.set('finished scan')
    insertFolderNodeFunc(tree1)
    

def openFolder():
    global p
    rootPath = filedialog.askdirectory(title = "Select Folder to open")
    rootPath = rootPath.replace('/',os.sep)
    print(rootPath)
    recursive_listdirThread(tree1,rootPath)

rowID = ''
def click_tree(event):
    global rowID
    m = Menu(root, tearoff=0)
    m.add_command(label="Go to folder",command=goToFolder)
    try:
        rowID = tree1.identify('item', event.x, event.y)
        print('click item',tree1.item(rowID)["values"][1])
        m.tk_popup(event.x_root, event.y_root)
    finally:
        m.grab_release()

def goToFolder():
    global rowID
    print(tree1.item(rowID)["values"][1])
    os.startfile(tree1.item(rowID)["values"][1])

if __name__ == '__main__':
    root = Tk()
    root.geometry('1024x768+888+444')
    
    scr1=Scrollbar(root)
    scr1.pack(fill=Y,side=RIGHT)
    scr2=Scrollbar(orient=HORIZONTAL)
    scr2.pack(fill=X,side=BOTTOM)
    
    
    tree1 = Treeview(root, columns=('#1','#2')) 
    tree1.column('#0', width=120, anchor=W)
    tree1.column('#1', width=90, anchor=CENTER)
    tree1.column('#2', width=90, anchor=CENTER)

    tree1.heading('#0', text='名称')
    tree1.heading('#1', text='大小')
    tree1.heading('#2', text='路径')


    tree1.pack(fill=BOTH,expand=True)
    
    tree1.config(xscrollcommand = scr2.set)
    scr2.config(command =tree1.xview)
    
    tree1.config(yscrollcommand = scr1.set)
    scr1.config(command = tree1.yview)

    tree1.bind("<Button-3>",click_tree)
    

    menubar = Menu(root)
    filemenu = Menu(menubar, tearoff=0)
    filemenu.add_command(label="Open", command=openFolder)

    filemenu.add_separator()

    filemenu.add_command(label="Exit", command=root.quit)
    menubar.add_cascade(label="File", menu=filemenu)
    editmenu = Menu(menubar, tearoff=0)
    editmenu.add_command(label="Undo", command=openFolder)

    editmenu.add_separator()

    editmenu.add_command(label="Cut", command=openFolder)

    menubar.add_cascade(label="Edit", menu=editmenu)
    helpmenu = Menu(menubar, tearoff=0)
    helpmenu.add_command(label="Help Index", command=openFolder)
    helpmenu.add_command(label="About...", command=openFolder)
    menubar.add_cascade(label="Help", menu=helpmenu)

    root.config(menu=menubar)

    statusvar = StringVar()
    statusvar.set("Ready")
    statusbar = Label(root, textvariable=statusvar, relief=SUNKEN, anchor=W)

    statusbar.pack(side=BOTTOM, fill=X)
    root.mainloop()
