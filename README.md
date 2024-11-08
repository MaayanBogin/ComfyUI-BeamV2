
# Guide: Running ComfyUI Workflows on Beam

## Introduction

This guide demonstrates how to execute workflows created with [ComfyUI](https://github.com/comfyanonymous/ComfyUI) on the Beam cloud service. ComfyUI is a powerful and modular stable diffusion GUI and backend that allows users to design and execute advanced stable diffusion pipelines 

To facilitate the transition from ComfyUI's visual workflows to executable Python code, we'll utilize the [ComfyUI-to-Python-Extension](https://github.com/pydn/ComfyUI-to-Python-Extension). This tool translates ComfyUI workflows into Python scripts, enabling seamless integration with Beam. :contentReference[oaicite:1]{index=1}


## Prerequisites

Before proceeding, ensure you have the following:

- **ComfyUI**: A working installation of ComfyUI.
- **ComfyUI-to-Python-Extension**: Access to this repository for converting workflows to Python scripts. [GitHub Repository](https://github.com/pydn/ComfyUI-to-Python-Extension)

## Step 1: Designing Your Workflow in ComfyUI

1. **Create Your Workflow**: Use the node-based interface to design your desired pipeline.
2. **Install the ComfyUI-to-Python-Extension**:
   - To enable script export options, install the ComfyUI-to-Python-Extension in your `ComfyUI/custom_nodes` directory:
     ```bash
     cd ComfyUI/custom_nodes
     git clone https://github.com/pydn/ComfyUI-to-Python-Extension.git
     cd ComfyUI-to-Python-Extension
     pip install -r requirements.txt
     ```
3. **Enable Developer Mode**:
   - Click the gear icon above the "Queue Prompt" button.
   - Check the "Enable Dev mode Options" box.

4. **Export the Workflow**:
   - With Developer Mode enabled, you can use two options:
     - **Save (API Format)**: Saves the workflow as a JSON file (e.g., `workflow_api.json`), which you can later convert manually using the ComfyUI-to-Python-Extension.
     - **Save as Script**: This newer option directly exports the workflow to a Python script, streamlining the conversion process.(If this works for you, you may skip step 3)

** Note that this feature may not work for all users or capture every parameter, so verify the output if you encounter issues.

## Step 2: Converting the Workflow to a Python Script 

3. **Convert the Workflow (if needed)**:
   - If you used the JSON export, convert it to a Python script:
     ```bash
     python comfyui_to_python.py --input_file path/to/workflow_api.json --output_file my_workflow.py
     ```
   - **Note**: The generated script may miss some node parameters or custom paths. Reviewing the output and comparing it with your `workflow.json` is recommended.

## Step 3: Preparing for Deployment on Beam

For Beam deployment, you need to adjust your script's paths and organize resources. Here’s a structured approach:

### Configuring Paths in `files_paths.py`

In ComfyUI, resource paths are defined in `files_paths.py`, which organizes directories for temporary files, user files, and—most importantly—model files. For Beam, update these paths to use volumes:

1. **Model and Resource Paths**:
   - You can define paths in Beam’s volume directory or, alternatively, store some files in temporary paths to avoid persistence. Here’s an example setup:
     ```python
     import os

     base_path = os.path.dirname(os.path.realpath(__file__))
     volume_dir = "./models"  # Beam volume directory for models
     temp_dir = "/tmp"  # Temporary directory path

     # Configure resource paths
     models_dir = os.path.join(volume_dir, "your-model-path-here")
     folder_names_and_paths = {
         "checkpoints": [os.path.join(models_dir, "checkpoints")],
         "loras": [os.path.join(models_dir, "loras")],
         "vae": [os.path.join(temp_dir, "vae")],  # Temporary directory for VAEs
         # Add other resources as needed
     }
     ```

   - **Tip**: You can use Beam volumes to persist model files across sessions. For temporary files, like generated outputs or intermediate files, store them in paths like `/tmp` to avoid unnecessary storage.

2. **Updating Custom Paths**:
   - Some nodes in ComfyUI require custom paths, often specified in the node’s script or `__init__.py`. Adjust these paths to point to Beam volumes or temporary storage as needed. Check `files_paths.py` for any hardcoded paths that may need updating.

3. **Optimize Workflow Size**:
   - ComfyUI initializes from scratch each time it runs. Keeping your workflow lean—especially by reducing the number of nodes—improves startup efficiency on Beam.

## Step 4: Deploying on Beam

With paths configured, you’re ready to deploy your workflow on Beam.

1. **Create an `app.py` File**:
   - In your project directory, create a file named `app.py` that will serve as your main entry point for Beam deployment:
     ```python
     from beam import Image, Volume, endpoint, Output
     from my_workflow import run_workflow  # Import your workflow function

     # Define the endpoint with required resources
      @task_queue(
          name="ComfyUI",
          cpu=4,
          memory="32Gi",
          gpu="A10G",
          image=Image(
              python_version="python3.10",
              python_packages="requirements.txt",
              # commands=["pip3 install opencv-python"],
          ),
          volumes=[Volume(name="PUTVOLUMEHERE", mount_path="./VOLUMEPATH")],
      )
      def generate_video(**inputs):
          main()
     ```

2. **Deploy to Beam**:
   - Use the Beam CLI to deploy your application:
     ```bash
     beam deploy app.py:generate 
     ```
## Additional Resources

- **ComfyUI-to-Python-Extension**: A tool to convert ComfyUI workflows into Python scripts. [GitHub Repository](https://github.com/pydn/ComfyUI-to-Python-Extension)
- **ComfyUI-BeamV2**: An example template of ComfyUI with dummy scripts for Beam deployment. [GitHub Repository](https://github.com/MaayanBogin/ComfyUI-BeamV2)


