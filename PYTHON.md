# Python

## Poetry Project pulled from Github
- pull the git repo: `git clone https://github.com/delimarr/simon_demo.git`
- cd non_poetry_project
- `poetry init`  # Only if pyproject.toml does not exist
- `poetry install`  # After answering prompts
- create new python 3.12 environment: `python3.12 -m venv name_env`
- active the newly create environment: `.\name_env\Scripts\activate` or mac `source name_env/Scripts/activate`
- install poetry: `pip install poetry`
- navigate to the folder, containing your `pyproject.toml`
- install your python dependencies and package, run: `poetry install`

# New Poetry-Project
- create new python 3.12 environment: `python3.12 -m venv name_env`
- active the newly create environment: `.\name_env\Scripts\activate` or mac `source name_env/Scripts/activate`
- install poetry: `pip install poetry`
- create poetry: `poetry new my_project`
- `cd my_project`
- (Optional)`poetry env use python3.12`  # Or whichever version you want
- (Optional)To add a dependency:`poetry add requests` 
- (Optional)To add a dev dependency:`poetry add --dev black`
- Open the virtual environment shell: `poetry shell`
- To run Python commands: `python my_project/__init__.py`

## Adding Libarys
- `cd your_project`
- `poetry add pandas` the libary is added in pyproject.toml
- Update Libary: `poetry update pandas`
- Removing Libary: `poetry remove pandas`

# Standalone Application
- cd to main.py folder
- `pip install pyinstaller`
  
-  `pyinstaller`command
- `main.py`Entry script
- `--noconsole`No terminal or console
- `--windowed`or `-w`supresses terminal (similar than --noconsole)
- `--name nameApp`Name of the App
- `--add-data "source_path:destination_path"`additiononal folder and files (on Windows ; instead of : )
- `--hidden-import=...`import modules manualy that pyinstalle might not find (tkinter, cv2, PIL, ...)
- `--onefile` bundle everything in one file. On Mac not with `--noconsole`or `--windowed`

### With Poetry
- `poetry run pyinstaller [your options here]` Options like above. Like this pyinstaller should find Modules/Dependencies
- `poetry add --dev pyinstaller`to add pyinstaller temporaly (not production)


# Main File
- `if __name__ == "__main__":`
- Folder that contain parts of the program should contain a ` __init__.py `
 file.
