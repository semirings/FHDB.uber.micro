An explainer video that demonstrates the fhdb stack.

F -- FHIR
H -- HAZoo
D -- D4M
B -- Blender

# Conops
FHIR resources are stored in HAZoo in D4M/AA table pairs.
D4M queries HAZoo producing a result in AA format.
The AA is saved to the file system as a CSV file.
The CSV file is posted to the Blender ms which uses Blender to display the data.

# Explaination

## logline
Fly through massive sets of clinical data in an immersive display.

## synopsis
For people who have a need
A user using Pluto.jl or Jupyter notebook writes a D4M query.  The query is posted from within the notebook to HAZoo.  HAZoo searches the data store assembling an AA, which it returns to the notebook.  The notebook writes the AA to  a temp CSV file.  The notebook then posts the CSV file to BlenderMS.  BlenderMS then executes in the Blender API to display the data from the CSV in the the Blender environment.

## treatment


### key concepts:
- D4M as a DSL
- HAZoo can handle large datasets
- AA as an organizing paradigm. 