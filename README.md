# Training Tesseract 5 in Docker

This guide provides step-by-step instructions for training Tesseract 5 in a Docker container. Docker allows you to create a reproducible environment for training Tesseract OCR models. By following the steps outlined below, you can set up a Docker container with Ubuntu, install Tesseract 5 and the necessary training tools, obtain training data, organize the data, and start the training process.

## Create Ubuntu container

1. Open the terminal.

2. Pull the Ubuntu Docker image:

   ```shell
   docker pull ubuntu
   ```

   If you are interested in a specific version, you can specify it:

   ```shell
   docker pull ubuntu:22.04
   ```

3. Run the Docker image:

   ```shell
   docker run -ti --rm ubuntu /bin/bash
   ```

   Note: By default, the Docker Ubuntu image does not have the `lsb_release` command available. You can use the `cat` command to check the OS information instead.

4. Check the OS version:

   ```shell
   cat /etc/os-release
   ```

   If the `lsb-release` package is not installed, update the package sources and install it:

   ```shell
   apt update && apt install lsb-core
   ```

   Verify the OS version again:

   ```shell
   lsb_release -a
   ```

5. Create a shared directory between your host system and the Docker container:
   In the container's terminal, create a directory named `Docker_Share`:

   ```shell
   mkdir -p Docker_Share
   ```

   Verify that the directory was created:

   ```shell
   ls
   ```

6. In a separate terminal on your host machine, check the current running container ID:

   ```shell
   docker ps
   ```

   Make note of the container ID.

7. Save the Docker container state as a new image:

   ```shell
   docker commit -p container_id new_image_name
   ```

   For example:

   ```shell
   docker commit -p 3409ehfu384f myubuntu
   ```

   Replace `container_id` with the ID of the container obtained in the previous step, and `new_image_name` with the desired name for the new image.

9. Verify that the new image was created:

   ```shell
   docker images
   ```

10. Stop the Docker container:

    ```shell
    docker stop container_id
    ```

    Replace `container_id` with the ID of the container obtained earlier.

11. Restart the container with the shared data:

    ```shell
    docker run -ti -v /host/machine/dir:/Docker_Share image_name /bin/bash
    ```

    For example:

    ```shell
    docker run -ti -v C:\training_data:/Docker_Share myubuntu /bin/bash
    ```

    Replace `/host/machine/dir` with the directory path on your host machine that you want to share with the container, `image_name` with the name of the new image created in the previous step, and `/bin/bash` to start the container with a terminal.

## Install Tesseract 5 in the container

1. In the container's terminal, update the package sources and install Git:

   ```shell
   apt update && apt install git
   ```

2. Clone the Tesseract repository:

   ```shell
   git clone https://github.com/tesseract-ocr/tesseract.git
   ```

   Verify that the `tesseract` directory was created:

   ```shell
   ls
   ```

3. Install auxiliary libraries required for Tesseract:

   ```shell
   apt update && apt install autoconf automake libtool pkg-config libpng-dev libjpeg8-dev libtiff5-dev zlib1g-dev libwebpdemux2 libwebp-dev libopenjp2-7-dev libgif-dev libarchive-dev libcurl4-openssl-dev libicu-dev libpango1.0-dev libcairo2-dev libleptonica-dev
   ```

4. Navigate to the `/tesseract` directory:

   ```shell
   cd /tesseract
   ```

5. Run the `autogen.sh` script:

   ```shell
   ./autogen.sh
   ```

6. Run the `configure` script:

   ```shell
   ./configure
   ```

7. Build and install Tesseract OCR 5:

   ```shell
   make
   make install
   ldconfig
   ```

8. Install the Tesseract training tools:

   ```shell
   make training
   make training-install
   ```

9. Clone the `tesstrain` repository:

   ```shell
   git clone https://github.com/tesseract-ocr/tesstrain.git
   ```

10. Navigate to the `tesstrain` directory:

    ```shell
    cd /tesseract/tesstrain
    ```

11. Install `wget` and the required Python libraries:

    ```shell
    apt update && apt install wget python3-pip
    pip install -r requirements.txt
    ```

12. Fetch language data:

    ```shell
    make tesseract-langdata
    ```

## Get Training Data

To train a Tesseract OCR model, you need the following training data:

- [lang].[font].exp[number].tif (line string image file)
- [lang].[font].exp[number].gt.txt (ground truth text file)

For example:

- chi_tra.DFKai.exp0.tif
- chi_tra.DFKai.exp0.gt.txt

Optional training data includes:

- [lang].[font].exp[number].box

The `.box` files contain information about character positions in the image, improving the training process and model accuracy.

Move all the training data into the directory shared with the Docker container. For example, if your shared directory on the host machine is `C:\training_data`, place all the `.gt.txt`, `.tif`, and `.box` files in that directory.

## Organize Training Data

1. Copy the training data from the shared directory to the appropriate location:

   ```shell
   cp -r /Docker_Share /tesseract/tesstrain/data/[lang].[font]-ground-truth
   ```

   Replace `[lang].[font]` with the appropriate language and font information.

2. Download the traineddata files you need from the [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) repository. Make sure to download the `eng.traineddata` file for any language you are training. For example, if you are training Chinese Traditional (chi_tra), download the `chi_tra.traineddata` file.

3. Move the downloaded traineddata files into the shared directory. For example, move `eng.traineddata` and `chi_tra.traineddata` to `C:\training_data` on the host machine.

4. Move the traineddata files to the default training directory:

   ```shell
   mv /Docker_Share/*.traineddata /usr/local/share/tessdata/
   ```

   Now your training data is organized and ready for training the new model.

## Start training

1. Navigate to the training directory:

   ```shell
   cd /tesseract/tesstrain
   ```

2. If you have .box files and want to avoid overwriting them during the training process, modify the Makefile:

   ```shell
   apt update && apt install nano
   cd /tesseract/tesstrain
   nano Makefile
   ```

   Locate the lines starting with `%.box` and comment them out.

   Original lines:

   ```shell
   %.box: %.png %.gt.txt
       PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.png" -t "$*.gt.txt" > "$@"

   %.box: %.bin.png %.gt.txt
       PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.bin.png" -t "$*.gt.txt" > "$@"

   %.box: %.nrm.png %.gt.txt
       PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.nrm.png" -t "$*.gt.txt" > "$@"

   %.box: %.raw.png %.gt.txt
       PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.raw.png" -t "$*.gt.txt" > "$@"

   %.box: %.tif %.gt.txt
       PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.tif" -t "$*.gt.txt" > "$@"
   ```

   Modified lines:

   ```shell
   # %.box: %.png %.gt.txt
   #    PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.png" -t "$*.gt.txt" > "$@"

   # %.box: %.bin.png %.gt.txt
   #    PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.bin.png" -t "$*.gt.txt" > "$@"

   # %.box: %.nrm.png %.gt.txt
   #    PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.nrm.png" -t "$*.gt.txt" > "$@"

   # %.box: %.raw.png %.gt.txt
   #    PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.raw.png" -t "$*.gt.txt" > "$@"

   # %.box: %.tif %.gt.txt
   #    PYTHONIOENCODING=utf-8 $(PY_CMD) $(GENERATE_BOX_SCRIPT) -i "$*.tif" -t "$*.gt.txt" > "$@"
   ```

   Press `Ctrl + O` and then `Enter` to save the modified Makefile. Press `Ctrl + X` to exit the editor.

4. Start training a new model:

   ```shell
   make training MODEL_NAME=[lang].[font] TESSDATA=/usr/local/share/tessdata
   ```

   Replace `[lang].[font]` with the appropriate language and font information.

5. If you want to fine-tune an existing model, use the `START_MODEL` parameter:

   ```shell
   make training MODEL_NAME=[lang].[font] START_MODEL=[lang] TESSDATA=/usr/local/share/tessdata
   ```

   Replace `[lang].[font]` with the appropriate language and font information.

6. After training, you can find the traineddata of the new model in the default output path:

   ```shell
   cd /tesseract/tesstrain/data/[lang].[font]
   ls
   ```

   Replace `[lang].[font]` with the appropriate language and font information.

7. Copy the traineddata of the new model to the shared directory:

   ```shell
   cp /tesseract/tesstrain/data/[lang].[font]/[lang].[font].traineddata /Docker_Share
   ```

   Replace `[lang].[font]` with the appropriate language and font information.

The traineddata file will now be available in the shared directory on your host machine.

## Reference

For detailed steps and additional information, please refer to the following resources:

- [How to Run Ubuntu as a Docker Container](https://www.makeuseof.com/run-ubuntu-docker-container/)
- [Compilation guide for various platforms | tessdoc](https://tesseract-ocr.github.io/tessdoc/Compiling.html)
- [GitHub - tesseract-ocr/tesstrain: Train Tesseract LSTM with make](https://github.com/tesseract-ocr/tesstrain)
