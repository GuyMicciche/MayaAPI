# Maya Plugin Types Guide

## 1. registerCommand - Commands/Operations

### When to use
Create custom tools, operations, or batch processes that users can execute.

### Examples
- Modeling operations: polySplitRing, polyBevel
- Scene utilities: deleteUnusedNodes, optimizeScene
- Custom workflows: ndDecimate, ndWeld
- Batch operations: exportMultipleObjects

### Template Example
```python
import maya.api.OpenMaya as OpenMaya

class MyCommandCmd(OpenMaya.MPxCommand):
    kCommandName = "myCommand"
    
    def __init__(self):
        OpenMaya.MPxCommand.__init__(self)
        self.mIsRedoable = False
        self.mValue = 0.0
        self.mName = ""
    
    @staticmethod
    def creator():
        return MyCommandCmd()
    
    @staticmethod
    def syntaxCreator():
        syntax = OpenMaya.MSyntax()
        syntax.addFlag("-v", "-value", OpenMaya.MSyntax.kDouble)
        syntax.addFlag("-n", "-name", OpenMaya.MSyntax.kString)
        syntax.addFlag("-h", "-help", OpenMaya.MSyntax.kNoArg)
        syntax.enableQuery(True)
        syntax.enableEdit(True)
        return syntax
    
    def doIt(self, argList):
        try:
            argDatabase = OpenMaya.MArgDatabase(self.syntax(), argList)
            if argDatabase.isFlagSet("-help"):
                OpenMaya.MGlobal.displayInfo("MyCommand usage: myCommand -value 1.0 -name 'test'")
                return
            self.mValue = 1.0
            self.mName = "default"
            if argDatabase.isFlagSet("-value"):
                self.mValue = argDatabase.flagArgumentDouble("-value", 0)
            if argDatabase.isFlagSet("-name"):
                self.mName = argDatabase.flagArgumentString("-name", 0)
            self.mIsRedoable = True
            self.redoIt()
        except Exception as e:
            OpenMaya.MGlobal.displayError(f"Command failed: {str(e)}")
            raise
    
    def redoIt(self):
        OpenMaya.MGlobal.displayInfo(f"Executed with value: {self.mValue}, name: {self.mName}")
    
    def undoIt(self):
        OpenMaya.MGlobal.displayInfo("Command undone")
    
    def isUndoable(self):
        return self.mIsRedoable

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerCommand(
        MyCommandCmd.kCommandName, 
        MyCommandCmd.creator,
        MyCommandCmd.syntaxCreator
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterCommand(MyCommandCmd.kCommandName)
```

---

## 2. registerNode - Data Processing/Computation

### When to use
Create nodes that process data, perform calculations, or maintain attribute relationships.

### Examples
- Math operations: multiplyDivide, plusMinusAverage
- Utility nodes: distanceBetween, vectorProduct
- Custom constraints: aimConstraint, pointConstraint
- Procedural generators: noise, fractal

### Template Example
```python
import maya.api.OpenMaya as OpenMaya

class MyNodeNode(OpenMaya.MPxNode):
    kTypeName = "myNode"
    kTypeId = OpenMaya.MTypeId(0x00000001)
    
    aInput = None
    aOutput = None
    aMultiply = None
    
    def __init__(self):
        OpenMaya.MPxNode.__init__(self)
    
    @staticmethod
    def creator():
        return MyNodeNode()
    
    @staticmethod
    def initialize():
        nAttr = OpenMaya.MFnNumericAttribute()
        MyNodeNode.aInput = nAttr.create("input", "in", OpenMaya.MFnNumericData.kFloat, 0.0)
        nAttr.keyable = True    # default: False — shows in Channel Box and can be keyed
        nAttr.storable = True   # default: True  — saved to .ma/.mb scene file
        nAttr.writable = True   # default: True  — can receive incoming connections
        nAttr.readable = True   # default: True  — can drive outgoing connections
        OpenMaya.MPxNode.addAttribute(MyNodeNode.aInput)

        MyNodeNode.aMultiply = nAttr.create("multiply", "mult", OpenMaya.MFnNumericData.kFloat, 2.0)
        nAttr.keyable = True    # default: False — shows in Channel Box and can be keyed
        nAttr.storable = True   # default: True  — saved to .ma/.mb scene file
        nAttr.writable = True   # default: True  — can receive incoming connections
        nAttr.readable = True   # default: True  — can drive outgoing connections
        OpenMaya.MPxNode.addAttribute(MyNodeNode.aMultiply)

        MyNodeNode.aOutput = nAttr.create("output", "out", OpenMaya.MFnNumericData.kFloat, 0.0)
        nAttr.keyable = False   # default: False — output attrs are not keyable
        nAttr.storable = False  # default: True  — output is computed, no need to store
        nAttr.writable = False  # default: True  — output cannot receive connections
        nAttr.readable = True   # default: True  — output can drive other nodes
        OpenMaya.MPxNode.addAttribute(MyNodeNode.aOutput)

        OpenMaya.MPxNode.attributeAffects(MyNodeNode.aInput, MyNodeNode.aOutput)
        OpenMaya.MPxNode.attributeAffects(MyNodeNode.aMultiply, MyNodeNode.aOutput)
    
    def compute(self, plug, dataBlock):
        if plug == MyNodeNode.aOutput:
            inputValue = dataBlock.inputValue(MyNodeNode.aInput).asFloat()
            multiplyValue = dataBlock.inputValue(MyNodeNode.aMultiply).asFloat()
            result = inputValue * multiplyValue
            outputHandle = dataBlock.outputValue(MyNodeNode.aOutput)
            outputHandle.setFloat(result)
            dataBlock.setClean(plug)
        else:
            return OpenMaya.kUnknownParameter

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerNode(
        MyNodeNode.kTypeName,
        MyNodeNode.kTypeId,
        MyNodeNode.creator,
        MyNodeNode.initialize,
        OpenMaya.MPxNode.kDependNode
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterNode(MyNodeNode.kTypeId)
```

---

## 3. registerShape - Custom Geometry/Viewport Objects

### When to use
Create custom drawable objects, locators, or specialized geometry types.

### Examples
- Locators: locator, annotationShape
- Guides: cameraShape, lightShape
- Custom geometry: nurbsCurve, mesh
- Visualization aids: measuring tools, reference objects

### Template Example
```python
import maya.api.OpenMaya as OpenMaya
import maya.api.OpenMayaUI as OpenMayaUI
import maya.api.OpenMayaRender as OpenMayaRender

class MyShapeNode(OpenMayaUI.MPxSurfaceShape):
    kTypeName = "myShape"
    kTypeId = OpenMaya.MTypeId(0x00000002)
    kDrawClassification = "drawdb/geometry/myShape"
    kDrawRegistrantId = "myShapePlugin"
    
    aSize = None
    aColor = None
    
    def __init__(self):
        OpenMayaUI.MPxSurfaceShape.__init__(self)
    
    @staticmethod
    def creator():
        return MyShapeNode()
    
    @staticmethod
    def initialize():
        nAttr = OpenMaya.MFnNumericAttribute()
        MyShapeNode.aSize = nAttr.create("size", "sz", OpenMaya.MFnNumericData.kFloat, 1.0)
        nAttr.keyable = True    # default: False — shows in Channel Box and can be keyed
        nAttr.storable = True   # default: True  — saved to .ma/.mb scene file
        nAttr.writable = True   # default: True  — can receive incoming connections
        OpenMaya.MPxNode.addAttribute(MyShapeNode.aSize)

        MyShapeNode.aColor = nAttr.createColor("color", "clr")
        nAttr.keyable = True    # default: False — shows in Channel Box and can be keyed
        nAttr.storable = True   # default: True  — saved to .ma/.mb scene file
        nAttr.writable = True   # default: True  — can receive incoming connections
        nAttr.default = (1.0, 0.0, 0.0)
        OpenMaya.MPxNode.addAttribute(MyShapeNode.aColor)
    
    def isBounded(self):
        return True
    
    def boundingBox(self):
        size = 1.0
        try:
            sizePlug = OpenMaya.MPlug(self.thisMObject(), MyShapeNode.aSize)
            size = sizePlug.asFloat()
        except:
            pass
        bbox = OpenMaya.MBoundingBox()
        bbox.expand(OpenMaya.MPoint(-size, -size, -size))
        bbox.expand(OpenMaya.MPoint(size, size, size))
        return bbox

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerShape(
        MyShapeNode.kTypeName,
        MyShapeNode.kTypeId,
        MyShapeNode.creator,
        MyShapeNode.initialize,
        None,
        MyShapeNode.kDrawClassification
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterNode(MyShapeNode.kTypeId)
```

---

## 4. registerData - Custom Data Types

### When to use
Create custom data structures stored, passed between nodes, or cached.

### Examples
- Complex data: point clouds, mesh data
- Cached computations: simulation states, optimization results
- Custom formats: proprietary file data
- Inter-node communication

### Template Example
```python
import maya.api.OpenMaya as OpenMaya

class MyDataData(OpenMaya.MPxData):
    kTypeName = "myData"
    kTypeId = OpenMaya.MTypeId(0x00000003)
    
    def __init__(self):
        OpenMaya.MPxData.__init__(self)
        self.mValue = 0.0
        self.mName = ""
        self.mValues = []
    
    @staticmethod
    def creator():
        return MyDataData()
    
    def typeId(self):
        return MyDataData.kTypeId
    
    def name(self):
        return MyDataData.kTypeName
    
    def copy(self, other):
        if isinstance(other, MyDataData):
            self.mValue = other.mValue
            self.mName = other.mName
            self.mValues = other.mValues[:]
    
    def readASCII(self, argList, lastParsedElement):
        if lastParsedElement < len(argList):
            try:
                self.mValue = float(argList[lastParsedElement])
                lastParsedElement += 1
                if lastParsedElement < len(argList):
                    self.mName = argList[lastParsedElement]
                    lastParsedElement += 1
            except (ValueError, IndexError):
                pass
        return lastParsedElement
    
    def writeASCII(self):
        return f"{self.mValue} {self.mName}"

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerData(
        MyDataData.kTypeName,
        MyDataData.kTypeId,
        MyDataData.creator
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterData(MyDataData.kTypeId)
```

---

## 5. registerContextCommand - Interactive Tools

### When to use
Create interactive viewport tools responding to mouse/keyboard input.

### Examples
- Modeling tools: polyCreateFacet, polySplit
- Paint tools: artAttrPaint, sculptMesh
- Measurement tools: distanceTool, angleTool
- Custom manipulators: selection, transformation tools

### Template Example
```python
import maya.api.OpenMaya as OpenMaya
import maya.api.OpenMayaUI as OpenMayaUI

class MyContextCommandCmd(OpenMayaUI.MPxContextCommand):
    kCommandName = "myContextCommand"
    
    def __init__(self):
        OpenMayaUI.MPxContextCommand.__init__(self)
        self.mContext = None
    
    @staticmethod
    def creator():
        return MyContextCommandCmd()
    
    @staticmethod
    def syntaxCreator():
        syntax = OpenMaya.MSyntax()
        syntax.addFlag("-r", "-radius", OpenMaya.MSyntax.kDouble)
        syntax.addFlag("-c", "-color", OpenMaya.MSyntax.kDouble, OpenMaya.MSyntax.kDouble, OpenMaya.MSyntax.kDouble)
        return syntax
    
    def makeObj(self):
        self.mContext = MyContextTool()
        return self.mContext

class MyContextTool(OpenMayaUI.MPxContext):
    def __init__(self):
        OpenMayaUI.MPxContext.__init__(self)
        self.mRadius = 1.0
    
    def setRadius(self, radius):
        self.mRadius = radius
    
    def getRadius(self):
        return self.mRadius

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerContextCommand(
        MyContextCommandCmd.kCommandName,
        MyContextCommandCmd.creator,
        MyContextCommandCmd.syntaxCreator
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterContextCommand(MyContextCommandCmd.kCommandName)
```

---

## 6. registerDragAndDropBehavior - UI Interactions

### When to use
Customize object behaviors when dragged and dropped in the UI.

### Examples
- Asset management: dragging textures onto materials
- Hierarchy operations: parent/child relationships
- Component assignment: dragging objects to sets/groups
- Custom workflows: specialized connection behaviors

### Template Example
```python
import maya.api.OpenMaya as OpenMaya

class MyDragAndDropBehavior(OpenMaya.MPxDragAndDropBehavior):
    kTypeName = "myDragAndDropBehavior"
    
    def __init__(self):
        OpenMaya.MPxDragAndDropBehavior.__init__(self)
    
    @staticmethod
    def creator():
        return MyDragAndDropBehavior()

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerDragAndDropBehavior(
        MyDragAndDropBehavior.kTypeName,
        MyDragAndDropBehavior.creator
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterDragAndDropBehavior(MyDragAndDropBehavior.kTypeName)
```

---

## 7. registerAttributePatternFactory - Dynamic Attributes

### When to use
Automatically create attributes based on patterns, conditions, or node types.

### Examples
- Procedural attributes: auto-generating animation controls
- Template systems: standardized attribute sets for rigs
- Dynamic UI: attributes based on other settings
- Workflow automation: consistent attribute creation

### Template Example
```python
import maya.api.OpenMaya as OpenMaya

class MyAttributePattern(OpenMaya.MPxAttributePattern):
    kPatternName = "myAttributePattern"
    
    def __init__(self):
        OpenMaya.MPxAttributePattern.__init__(self)
    
    @staticmethod
    def creator():
        return MyAttributePattern()

class MyAttributePatternFactory(OpenMaya.MPxAttributePatternFactory):
    kFactoryName = "myAttributePatternFactory"
    
    def __init__(self):
        OpenMaya.MPxAttributePatternFactory.__init__(self)
    
    @staticmethod
    def creator():
        return MyAttributePatternFactory()

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.registerAttributePatternFactory(
        MyAttributePatternFactory.kFactoryName,
        MyAttributePatternFactory.creator
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMaya.MFnPlugin(mobject)
    mPlugin.deregisterAttributePatternFactory(MyAttributePatternFactory.kFactoryName)
```

---

## 8. registerFileTranslator - File Import/Export

### When to use
Add support for reading or writing custom file formats in Maya's File > Import/Export dialogs.

### Examples
- Exporters: FBX, glTF, USD, Alembic
- Importers: custom game formats, proprietary data
- Converters: batch scene format conversion
- Pipeline tools: DCC interchange formats

### Template Example

**myTranslator.py**
```python
import maya.api.OpenMaya as OpenMaya
import maya.OpenMayaMPx as OpenMayaMPx

PLUGIN_NAME = "My Format"
FILE_EXT = "myf"

class MyTranslator(OpenMayaMPx.MPxFileTranslator):
    def __init__(self):
        OpenMayaMPx.MPxFileTranslator.__init__(self)

    @staticmethod
    def creator():
        return OpenMayaMPx.asMPxPtr(MyTranslator())

    @staticmethod
    def _parse_options(opt_string):
        """Parse semicolon-separated key=value option string into a dict."""
        options = {}
        for pair in opt_string.split(";"):
            if "=" in pair:
                key, value = pair.split("=", 1)
                options[key.strip()] = value.strip()
        return options

    def haveWriteMethod(self):
        return True  # supports export

    def haveReadMethod(self):
        return True  # supports import

    def filter(self):
        return "*.{}".format(FILE_EXT)  # file browser filter

    def defaultExtension(self):
        return FILE_EXT

    def writer(self, file_obj, opt_string, access_mode):
        # opt_string contains options passed from the options dialog, e.g. "scale=1.0;normals=1"
        path = file_obj.fullName()
        options = MyTranslator._parse_options(opt_string)
        try:
            # write your format here
            with open(path, "w") as f:
                f.write("# Exported by MyTranslator\n")
                f.write("scale={}\n".format(options.get("scale", "1.0")))
            OpenMaya.MGlobal.displayInfo("Exported: {}".format(path))
        except Exception as e:
            OpenMaya.MGlobal.displayError("Export failed: {}".format(str(e)))
            raise

    def reader(self, file_obj, opt_string, access_mode):
        path = file_obj.fullName()
        try:
            # read your format here
            with open(path, "r") as f:
                data = f.read()
            OpenMaya.MGlobal.displayInfo("Imported: {}".format(path))
        except Exception as e:
            OpenMaya.MGlobal.displayError("Import failed: {}".format(str(e)))
            raise

    def identifyFile(self, file_obj, buffer, size):
        # return kIsMyFileType if the file belongs to this translator
        if file_obj.fullName().endswith(".{}".format(FILE_EXT)):
            return OpenMayaMPx.MPxFileTranslator.kIsMyFileType
        return OpenMayaMPx.MPxFileTranslator.kNotMyFileType

# Registration

def initializePlugin(mobject):
    mPlugin = OpenMayaMPx.MFnPlugin(mobject)
    mPlugin.registerFileTranslator(
        PLUGIN_NAME,
        None,                   # icon file path (optional)
        MyTranslator.creator,
        "myTranslatorOpts",     # MEL proc name for the options dialog (optional)
        "scale=1.0;normals=1"   # default option string passed to writer/reader
    )

def uninitializePlugin(mobject):
    mPlugin = OpenMayaMPx.MFnPlugin(mobject)
    mPlugin.deregisterFileTranslator(PLUGIN_NAME)
```

**myTranslatorOpts.mel** — options dialog shown in Maya's Export dialog when the user clicks Options
```mel
// myTranslatorOpts.mel
// Called by Maya with $action = "post" (build UI) or "query" (read UI and return option string).
// The option string format is: "key1=value1;key2=value2;"
// This proc name must match the string passed to registerFileTranslator.

global proc int myTranslatorOpts(string $parent, string $action, string $initialSettings, string $resultCallback)
{
    int $result = 1;
    string $optionList[];
    string $optionBreakDown[];

    if ($action == "post") {
        setParent $parent;

        columnLayout -adjustableColumn true;

        checkBox -label "Export Normals" normalsCheck;
        floatField -value 1.0 scaleField;
        text -label "Scale";

        // Apply initial settings to the UI controls
        if (size($initialSettings) > 0) {
            tokenize($initialSettings, ";", $optionList);
            for ($i = 0; $i < size($optionList); $i++) {
                tokenize($optionList[$i], "=", $optionBreakDown);
                if ($optionBreakDown[0] == "normals") {
                    checkBox -edit -value ($optionBreakDown[1] == "1") normalsCheck;
                } else if ($optionBreakDown[0] == "scale") {
                    floatField -edit -value $optionBreakDown[1] scaleField;
                }
            }
        }

    } else if ($action == "query") {
        // Read UI controls and build the option string, then pass it back via the callback
        string $currentOptions = "";
        $currentOptions += "normals=" + `checkBox -q -value normalsCheck` + ";";
        $currentOptions += "scale="   + `floatField -q -value scaleField` + ";";
        eval($resultCallback + " \"" + $currentOptions + "\"");

    } else {
        $result = 0;
    }

    return $result;
}
```

---

# Quick Decision Guide

| Need to...             | Plugin Type                     | Description                                                          |
|------------------------|---------------------------------|----------------------------------------------------------------------|
| DO something           | registerCommand                 | Execute operations, batch processes, or tools via MEL/Python command |
| CALCULATE something    | registerNode                    | Process data and maintain attribute relationships in the DG          |
| DRAW something         | registerShape                   | Create custom geometry, locators, or viewport objects                |
| STORE complex data     | registerData                    | Define custom data types passed between nodes or cached              |
| INTERACT               | registerContextCommand          | Build interactive viewport tools driven by mouse/keyboard input      |
| DRAG/DROP behavior     | registerDragAndDropBehavior     | Customize what happens when objects are dragged in the UI            |
| AUTO-GENERATE attrs    | registerAttributePatternFactory | Dynamically create attribute sets based on patterns or conditions    |
| IMPORT/EXPORT files    | registerFileTranslator          | Add custom file format support to Maya's Import/Export dialogs       |

Most plugins start with registerCommand for basic operations, then evolve to registerNode for complex data processing or registerShape for custom viewport display.
