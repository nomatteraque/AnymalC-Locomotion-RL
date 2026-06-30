# Windows Setup Guide: Isaac Sim & Isaac Lab (Orbit)

This guide walks you through the step-by-step installation of **NVIDIA Isaac Sim** and **Isaac Lab** (the modern successor to Isaac Gym / Orbit) for reinforcement learning quadruped training on Windows 11 with an RTX 3070 Ti.

---

## Prerequisites & System Configuration

### 1. Update Graphics Drivers
Ensure you have the latest **NVIDIA Game Ready or Studio Driver** installed.
* Minimum version: `535+` (recommended: latest branch).
* Verify Vulkan and CUDA support are enabled.

### 2. Enable Windows Long Path Support
Many python and Omniverse directories exceed the standard Windows 260-character limit. You **must** enable long path support.
1. Press `Win + R`, type `regedit`, and hit **Enter**.
2. Navigate to: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem`.
3. Locate `LongPathsEnabled` and double-click it.
4. Set **Value data** to `1`.
5. Click **OK** and restart your PC to apply the registry changes.

---

## Step 1: Install NVIDIA Omniverse & Isaac Sim

Isaac Sim is distributed through the Omniverse Launcher.

1. **Download Omniverse Launcher**: Go to the [NVIDIA Omniverse website](https://www.nvidia.com/en-us/omniverse/) and download the **Omniverse Launcher for Windows**.
2. **Install and Log In**: Run the installer and sign in with your NVIDIA Developer account.
3. **Set Up Nucleus (Optional but recommended)**:
   * In the Omniverse Launcher, navigate to the **Nucleus** tab.
   * Click **Add Local Nucleus Service** and follow the prompts to install it. This hosts simulator assets locally for faster loading.
4. **Install Isaac Sim**:
   * Go to the **Exchange** tab in the Launcher.
   * Search for **Isaac Sim**.
   * Click on the latest version (e.g., `4.2.0` or `4.0.0` depending on release) and click **Install**.
   * Take note of the installation path (typically `C:\Users\<username>\AppData\Local\ov\pkg\isaac-sim-4.2.0`).

---

## Step 2: Set Up Conda & Isaac Lab Environment

Isaac Lab provides a script to automatically handle Conda environments and link to the simulator.

### 1. Install Miniconda or Anaconda
If you do not have Conda installed, download and install [Miniconda for Windows](https://docs.conda.io/en/latest/miniconda.html).

### 2. Clone the Isaac Lab Repository
Open a terminal (Command Prompt or PowerShell in Administrator mode) and clone the repository:
```cmd
git clone https://github.com/isaac-sim/IsaacLab.git
cd IsaacLab
```

### 3. Create the Conda Virtual Environment
Isaac Lab has a batch utility `isaaclab.bat` that simplifies setup.

Run the following command to create a conda environment named `env_isaaclab` containing the appropriate Python version (usually `3.10` or `3.11` depending on the Isaac Sim version):
```cmd
isaaclab.bat --conda env_isaaclab
```
*Note: This script will locate your Isaac Sim installation, create the conda environment, and link the simulator's Python libraries automatically.*

---

## Step 3: Install Learning Frameworks (rsl_rl)

For legged locomotion (quadruped walking/running), we use the **Robotic Systems Lab RL (`rsl_rl`)** framework, which is the official modern backend for legged Gym tasks.

1. Activate your newly created Conda environment:
   ```cmd
   conda activate env_isaaclab
   ```
2. Install the necessary reinforcement learning dependencies using the setup script:
   ```cmd
   isaaclab.bat --install rsl_rl
   ```
   *This command installs the `rsl_rl` library as well as other standard libraries like PyTorch, TensorBoard, and core dependencies into your environment.*

---

## Step 4: Verify the Setup

Let's run a test to ensure everything is connected and PyTorch is using your RTX 3070 Ti.

### 1. Basic Simulator Check
Test launching a simple visual simulation:
```cmd
:: From the IsaacLab directory
isaaclab.bat --play source/standalone/tutorials/00_sim_launch.py
```
A window should pop up showing a blank grid with the simulator running.

### 2. Verify PyTorch and CUDA
Check if PyTorch detects your GPU:
```cmd
python -c "import torch; print('CUDA Available:', torch.cuda.is_available()); print('Device Name:', torch.cuda.get_device_name(0))"
```
You should see:
```text
CUDA Available: True
Device Name: NVIDIA GeForce RTX 3070 Ti Laptop GPU
```

### 3. Run a Legged Quadruped Environment Test (Headless)
Run a training test with the default ANYmal C environment to verify RL configurations:
```cmd
python -m isaaclab.app --task Isaac-Velocity-Rough-Anymal-C-v0 --num_envs 64 --headless
```
If the environment successfully initializes, creates the 64 ANYmal C actors, and runs steps without errors, your setup is fully functional!

---

## Troubleshooting Common Windows Issues

* **Error: `ModuleNotFoundError: No module named 'isaacsim'`**
  * *Fix*: Make sure you activated your conda environment (`conda activate env_isaaclab`) before running commands, and that `isaaclab.bat --conda` linked the paths correctly.
* **Error: `Vulkan device not found`**
  * *Fix*: Ensure your monitor is plugged into the NVIDIA GPU (not integrated graphics) and that your graphics drivers are fully updated. If running over Remote Desktop, Vulkan might not be supported.
* **Error: Filename too long / Extraction failed**
  * *Fix*: Double check that you completed the **Long Path Support** registry step and restarted your PC.
