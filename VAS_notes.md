## Using Webcam in VAS

### Create a pipeline template;

For example, you can copy an existing pipeline.
```bash
cp -r pipelines/gstreamer/emotion_recognition pipelines/gstreamer/my_pipeline
```

Then you will have to change source element of this pipeline from *urisourcebin* to *v4l2src*.
```diff
3,4c3,4
< 	"template": "urisourcebin name=\"source\" ! concat name=c ! decodebin ! ...
< 	"description": "Emotion Recognition Pipeline",
---
> 	"template": "v4l2src name=\"source\" ! concat name=c ! decodebin ! ...
> 	"description": "My Pipeline",
```

### Start VAS with access to the webcam device file;

If you use docker/run.sh, make sure add priviledged option to docker.
```diff
@@ -232,6 +232,7 @@ if [ "${MODE}" == "DEV" ]; then
 elif [ "${MODE}" == "SERVICE" ]; then
   PORTS+="-p 8080:8080 "
   DEVICES+='-device /dev/dri '
+  PRIVILEGED="--privileged "
```

Then start serivce with /dev and your new pipeline volume.
```bash
./docker/run.sh -v /tmp:/tmp -v /dev:/dev --pipelines pipelines/gstreamer
```

3. Trigger pipeline with device /dev/video0

```bash
curl localhost:8080/pipelines/my_pipeline/1 -X POST -H 'Content-Type: application/json' -d '{ "source": { "path": "/dev/video0", "type": "device" }, "destination": { "type": "file", "path": "/tmp/results.txt", "format":"json-lines"}}'
```

4. Check /tmp/results.txt file, there shall be output.