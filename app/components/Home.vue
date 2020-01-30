<template>
	<Page @loaded="onLoaded">
        <ActionBar>
            <ActionItem icon="" text="Take Shot" @tap="onTakeShot" ios.position="left" />
        </ActionBar>
        <StackLayout>
            <Placeholder @creatingView="onCreatingView" id="placeholder-view"/>
        </StackLayout>
    </Page>
</template>

<script>
var permissions = require("nativescript-permissions");
var { alert } = require("ui/dialogs");
var app = require("@nativescript/core/application");
var page;

// Android
var mCameraId;
var mCaptureSession;
var mCameraDevice;
var mStateCallBack;
var mBackgroundHandler = null;
var mCameraOpenCloseLock = new java.util.concurrent.Semaphore(1);
var mTextureView;
var mSurfaceTexture;
var mPreviewRequestBuilder;
var mPreviewRequest;
var mImageReader;
var mCaptureCallback;
var mFlashSupported;
var mFile;
var mediaCodec;

var STATE_PREVIEW = 0;
var STATE_WAITING_LOCK = 1;
var STATE_WAITING_PRECAPTURE = 2;
var STATE_WAITING_NON_PRECAPTURE = 3;
var STATE_PICTURE_TAKEN = 4;
var mState = STATE_PREVIEW;

function runPrecaptureSequence() {
    // This is how to tell the camera to trigger.
    mPreviewRequestBuilder.set(android.hardware.camera2.CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER);
    // Tell #mCaptureCallback to wait for the precapture sequence to be set.
    mState = STATE_WAITING_PRECAPTURE;
    mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, mBackgroundHandler);
}

function startDecoding(surface) {
    if (mediaCodec != null) {
        return;
    }
    try {
        mediaCodec = android.media.MediaCodec.createDecoderByType("video/avc");
        console.log(mediaCodec)
    } catch (e) {
        console.log(e)
    }

}

function captureStillPicture() {
    // This is the CaptureRequest.Builder that we use to take a picture.
    var captureBuilder = mCameraDevice.createCaptureRequest(android.hardware.camera2.CameraDevice.TEMPLATE_STILL_CAPTURE);
    captureBuilder.addTarget(mImageReader.getSurface());
    console.log('capturing')

    // Use the same AE and AF modes as the preview.
    setAutoFlash(captureBuilder);

    var CaptureCallback = android.hardware.camera2.CameraCaptureSession.CaptureCallback.extend({
        onCaptureCompleted: function (session, request, result) {
            console.log("onCaptureCompleted");
            // console.log(mFile.toString());
        }
    });

    mCaptureSession.stopRepeating();
    mCaptureSession.abortCaptures();
    if (!mCaptureSession) {
        console.log('FAILED: NO CAPTURE SESSION');
        return;
    }
    mCaptureSession.capture(captureBuilder.build(), new CaptureCallback(), null);
}

function setAutoFlash(requestBuilder) {
    console.log("mFlashSupported in setAutoFlash:" + mFlashSupported);
    if (mFlashSupported) {
        requestBuilder.set(android.hardware.camera2.CaptureRequest.CONTROL_AE_MODE,
            android.hardware.camera2.CaptureRequest.CONTROL_AE_MODE.CONTROL_AE_MODE_ON_AUTO_FLASH);
    }
}

function createCameraPreviewSession() {
    console.log("createCameraPreviewSession");
    // console.log(mSurfaceTexture)

    if (!mSurfaceTexture || !mCameraDevice) {
        return;
    }

    var texture = mTextureView.getSurfaceTexture();

    // We configure the size of default buffer to be the size of camera preview we want.
    texture.setDefaultBufferSize(800, 480);

    // This is the output Surface we need to start preview.
    var surface = new android.view.Surface(texture);

    // // We set up a CaptureRequest.Builder with the output Surface.
    mPreviewRequestBuilder = mCameraDevice.createCaptureRequest(android.hardware.camera2.CameraDevice.TEMPLATE_PREVIEW);
    mPreviewRequestBuilder.addTarget(surface);

    var surfaceList = new java.util.ArrayList();
    surfaceList.add(surface);
    // console.log(surfaceList)
    mCameraDevice.createCaptureSession(surfaceList, new MyCameraCaptureSessionStateCallback(), null);
}

// from Java ; public static abstract class
var MyCameraCaptureSessionStateCallback = android.hardware.camera2.CameraCaptureSession.StateCallback.extend({
    onConfigured: function (cameraCaptureSession) {
        console.log("onConfigured " + cameraCaptureSession);

        if (mCameraDevice === null) {
            return;
        }

        mCaptureSession = cameraCaptureSession;

        // mPreviewRequestBuilder.set(android.hardware.camera2.CaptureRequest.CONTROL_AF_MODE);
        // Flash is automatically enabled when necessary.
        // setAutoFlash(mPreviewRequestBuilder);

        // Finally, we start displaying the camera preview.
        mPreviewRequest = mPreviewRequestBuilder.build();
        // mCaptureCallback = new MyCaptureSessionCaptureCallback();
        // mCaptureSession.setRepeatingRequest(mPreviewRequest, mCaptureCallback, null);
        const captureCallback = new MyCaptureSessionCaptureCallback();
        mCaptureSession.setRepeatingRequest(mPreviewRequest, captureCallback, null);

    },
    onConfigureFailed: function (cameraCaptureSession) {
        console.log("onConfigureFailed " + cameraCaptureSession);
    }
});

// from Java : public static abstract class
var MyCaptureSessionCaptureCallback = android.hardware.camera2.CameraCaptureSession.CaptureCallback.extend({
    process: function (result) {
        switch (mState) {
            case STATE_PREVIEW: {
                // startDecoding(mImageReader.getSurface())
                // We have nothing to do when the camera preview is working normally.
                break;
            }
            case STATE_WAITING_LOCK: {
                mCaptureSession.stopRepeating();
                // console.log(mTextureView.getBitmap())
                const image = mTextureView.getBitmap();
                startDecoding(image);
                // var afState = result.get(android.hardware.camera2.CaptureResult.CONTROL_AF_STATE);
                // if (afState === null) {
                //     captureStillPicture();
                // } else if (android.hardware.camera2.CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState ||
                //     android.hardware.camera2.CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED == afState) {
                //     console.log('test')
                //     // CONTROL_AE_STATE can be null on some devices
                //     var aeState = result.get(android.hardware.camera2.CaptureResult.CONTROL_AE_STATE);
                //     if (aeState === null ||
                //         aeState == android.hardware.camera2.CaptureResult.CONTROL_AE_STATE_CONVERGED) {
                //         mState = STATE_PICTURE_TAKEN;
                //         captureStillPicture();
                //     } else {
                //         captureStillPicture();
                //     }
                // } else {
                //     // console.log('set')
                // }
                break;
            }
            case STATE_WAITING_PRECAPTURE: {
                // CONTROL_AE_STATE can be null on some devices
                var aeStatee = result.get(android.hardware.camera2.CaptureResult.CONTROL_AE_STATE);
                if (aeStatee === null ||
                    aeStatee == android.hardware.camera2.CaptureResult.CONTROL_AE_STATE_PRECAPTURE ||
                    aeStatee == android.hardware.camera2.CaptureRequest.CONTROL_AE_STATE_FLASH_REQUIRED) {
                    mState = STATE_WAITING_NON_PRECAPTURE;
                }
                break;
            }
            case STATE_WAITING_NON_PRECAPTURE: {
                // CONTROL_AE_STATE can be null on some devices
                var aeStateee = result.get(android.hardware.camera2.CaptureResult.CONTROL_AE_STATE);
                if (aeStateee === null || aeStateee != android.hardware.camera2.CaptureResult.CONTROL_AE_STATE_PRECAPTURE) {
                    mState = STATE_PICTURE_TAKEN;
                    captureStillPicture();
                }
                break;
            }
        }
    },
    onCaptureProgressed: function (session, request, partialResult) {
        console.log("onCaptureProgressed");
        this.process(partialResult);
    },
    onCaptureCompleted: function (session, request, result) {
        // console.log("onCaptureCompleted");
        this.process(result);
    },
    onCaptureFailed: function (session, request, failure) {
        console.log("onCaptureFailed");
        console.log(failure);
    }
});

// from Java : public static abstract class
var MyStateCallback = android.hardware.camera2.CameraDevice.StateCallback.extend({
    onOpened: function (cameraDevice) {
        console.log("onOpened " + cameraDevice);

        mCameraOpenCloseLock.release();
        mCameraDevice = cameraDevice;
        createCameraPreviewSession();
    },
    onDisconnected: function (cameraDevice) {
        console.log("onDisconnected");

        mCameraOpenCloseLock.release();
        cameraDevice.close();
        mCameraDevice = null;
    },
    onError: function (cameraDevice, error) {
        console.log("onError");
        console.log("onError: device = " + cameraDevice);
        console.log("onError: error =  " + error);

        mCameraOpenCloseLock.release();
        cameraDevice.close();
        mCameraDevice = null;
    },
    onClosed: function (cameraDevice) {
        console.log("onClosed");
    }
});

// (example for: java static interface to javaScript )
// from Java : public static interface    
var mOnImageAvailableListener = new android.media.ImageReader.OnImageAvailableListener({
    onImageAvailable: function (reader) {

        // here we should save our image to file when image is available
        console.log("onImageAvailable");
        console.log(reader);
    }
});

// from Java : public static interface    
var mSurfaceTextureListener = new android.view.TextureView.SurfaceTextureListener({

    onSurfaceTextureAvailable: function (texture, width, height) {
        console.log('onSurfaceTextureAvailable');
        mSurfaceTexture = texture;
        createCameraPreviewSession();
        // openCamera(width, height);
    },

    onSurfaceTextureSizeChanged: function (texture, width, height) {
        console.log('onSurfaceTextureSizeChanged');
        // configureTransform(width, height);
    },

    onSurfaceTextureDestroyed: function (texture) {
        console.log("onSurfaceTextureDestroyed");
        return true;
    },

    onSurfaceTextureUpdated: function (texture) {
        // console.log("onSurfaceTextureUpdated");
    },

});

export default {
    data() {
        return {
            pictureFromCamera: null,
            textPicture: "Take a Picture"
        };
    },

    methods: {

        onTakeShot() {
            console.log('onTakeShot');
            mState = STATE_WAITING_LOCK;
            mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, mBackgroundHandler);
        },

        onLoaded(args) {
            page = args.object;
        },

        onCreatingView(args) {
            var appContext = app.android.context;

            var cameraManager = appContext.getSystemService(android.content.Context.CAMERA_SERVICE);
            var cameras = cameraManager.getCameraIdList();

            for (var index = 0; index < cameras.length; index++) {
                var currentCamera = cameras[index];
                var currentCameraSpecs = cameraManager.getCameraCharacteristics(currentCamera);

                var available = currentCameraSpecs.get(android.hardware.camera2.CameraCharacteristics.FLASH_INFO_AVAILABLE);
                mFlashSupported = available == null ? false : true;

                // get available lenses and set the camera-type (front or back)
                var facing = currentCameraSpecs.get(android.hardware.camera2.CameraCharacteristics.LENS_FACING);

                if (facing !== null && facing == android.hardware.camera2.CameraCharacteristics.LENS_FACING_BACK) {
                    console.log("BACK camera");
                    mCameraId = currentCamera;
                }

                // get all available sizes ad set the format
                var map = currentCameraSpecs.get(android.hardware.camera2.CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
                var format = map.getOutputSizes(android.graphics.ImageFormat.JPEG);
                console.log("Format: " + format + " " + format.length + " " + format[4]);

                // we are taking not the largest possible but some of the 5th in the list of resolutions
                if (format && format !== null) {
                    var dimensions = format[0].toString().split('x');
                    console.log(dimensions);
                    var largestWidth = +dimensions[0];
                    var largestHeight = +dimensions[1];

                    // set the output image characteristics
                    mImageReader = new android.media.ImageReader.newInstance(largestWidth, largestHeight, android.graphics.ImageFormat.JPEG, /*maxImages*/2);
                    mImageReader.setOnImageAvailableListener(mOnImageAvailableListener, mBackgroundHandler);
                }

            }

            mStateCallBack = new MyStateCallback();
            
            //API 23 runtime permission check
            android.os.Build.VERSION.SDK_INT < android.os.Build.VERSION_CODES.LOLLIPOP
                ? cameraManager.openCamera(mCameraId, mStateCallBack, mBackgroundHandler)
                : permissions
                    .requestPermission(
                        android.Manifest.permission.CAMERA,
                        "I need these permissions to use Android Camera"
                    )
                    .then(() => cameraManager.openCamera(mCameraId, mStateCallBack, mBackgroundHandler))
                    .catch(() => alert("Could not gain CAMERA permission. Please go to settings and turn on to use this app"));

            mTextureView = new android.view.TextureView(app.android.context);
            mTextureView.setSurfaceTextureListener(mSurfaceTextureListener);
            args.view = mTextureView;
        }
    }
};
</script>
