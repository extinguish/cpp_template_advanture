## About `cppTemplateAdventure`

首先我们来阐述一个问题,就是我们除了开发类库之外,还有在什么场合必须要使用`template`.
其中一个场合就是`std::shared_ptr<T>`和`多态`结合的情况下,我们必须要使用`template`.
考虑下面的场景,就是我们真实遇到的开发情况:

```cpp
//
// Created by guoshichao on 2020/5/22.
//

#include "jt1078/MediaStreamContext.h"

adasplus::MediaStreamContext::MediaStreamContext(const BYTE logicChannelNum) : mLogicChannelNum(logicChannelNum),
                                                                               mNetworkPerformer(nullptr),
                                                                               mMediaStreamStrategy(nullptr),
                                                                               mStreamType(CHILD_STREAM) {

}

adasplus::MediaStreamContext::~MediaStreamContext() {
    LOG808V("release the MediaStreamContext of channel [%d]", mLogicChannelNum);
    delete mMediaStreamStrategy;
    mMediaStreamStrategy = nullptr;
}

void adasplus::MediaStreamContext::setMediaStreamStrategy(adasplus::IMediaStreamStrategy *strategy) {
    this->mMediaStreamStrategy = strategy;
}

void adasplus::MediaStreamContext::setStreamingMode(adasplus::StreamingMode streamingMode) {
    IMediaStreamStrategy *mediaStreamStrategy = nullptr;
    switch (streamingMode) {
        case STREAMING_AV: {
            LOG808V("create new AVStreamStrategy of channel %.2x", mLogicChannelNum);
            mediaStreamStrategy = new AVStreamStrategy(mLogicChannelNum);
        }
            break;
        case STREAMING_VIDEO: {
            LOG808V("create new VideoStreamStrategy of channel %.2x", mLogicChannelNum);
            mediaStreamStrategy = new VideoStreamStrategy(mLogicChannelNum);
        }
            break;
        case STREAMING_TWO_WAY_INTERCOM: {
            LOG808V("create new TwoWayIntercomStreamStrategy of channel %.2x", mLogicChannelNum);
            mediaStreamStrategy = new TwoWayIntercomStreamStrategy(mLogicChannelNum);
        }
            break;
        case STREAMING_MONITOR: {
            LOG808V("create new MonitorStreamStrategy of channel %.2x", mLogicChannelNum);
            mediaStreamStrategy = new MonitorStreamStrategy(mLogicChannelNum);
        }
            break;
        case STREAMING_BROADCAST: {
            LOG808V("create new BroadcastStreamStrategy of channel %.2x", mLogicChannelNum);
            mediaStreamStrategy = new BroadcastStreamStrategy(mLogicChannelNum);
        }
            break;
        case STREAMING_PENETRATE:
            LOG808E("do not support this streaming mode of channel %.2x for now", mLogicChannelNum);
            break;
        default:
            LOG808E("FATAL STATE ERROR!!! channel %.2x the streaming mode unknown!!!", mLogicChannelNum);
            break;
    }
    if (mediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! channel [%d] CANNOT INIT LIVE STREAMING STRATEGY --> of mode %.2x",
                mLogicChannelNum, streamingMode);
        return;
    }
    mediaStreamStrategy->setupPhoneNum(mPhoneNum);
    mediaStreamStrategy->setNetworkPerformer(mNetworkPerformer);
    setMediaStreamStrategy(mediaStreamStrategy);
}

void adasplus::MediaStreamContext::setupPhoneNum(BYTE *phoneNum) {
    memcpy(mPhoneNum, phoneNum, 6);
}

void adasplus::MediaStreamContext::setNetworkPerformer(adasplus::INetworkPerformer *networkPerformer) {
    this->mNetworkPerformer = networkPerformer;
}

void adasplus::MediaStreamContext::onData(const uint8_t *data, const size_t dataLen) {
    // send this data to the related IMediaStreamStrategy
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! The MediaStreamingStrategy is not setup for action %s", __FUNCTION__);
        return;
    }
    mMediaStreamStrategy->receiveStreamFromMediaServer(data, dataLen);
}

void adasplus::MediaStreamContext::setStreamType(BYTE streamType) {
    this->mStreamType = streamType;
}

void adasplus::MediaStreamContext::closeStreaming(adasplus::CloseStreamingCmd closeType) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! Channel [%d] --> The MediaStreamingStrategy is not setup for action %s",
                mLogicChannelNum, __FUNCTION__);
        return;
    }
    switch (closeType) {
        case CLOSE_AUDIO_N_VIDEO:
            mMediaStreamStrategy->closeAVStreaming();
            break;
        case CLOSE_VIDEO_ONLY:
            mMediaStreamStrategy->closeVideoStreaming();
            break;
        case CLOSE_AUDIO_ONLY:
            mMediaStreamStrategy->closeAudioStreaming();
            break;
        default:
            LOG808E("FATAL ERROR!!! on channel [%d] --> Unknown close type %.2x", mLogicChannelNum, closeType);
            break;
    }
}

int adasplus::MediaStreamContext::controlStreaming(adasplus::AVTransmissionCmd cmd) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! The MediaStreamingStrategy is not setup for action %s", __FUNCTION__);
        return -1;
    }
    int ctrlRet = 0;
    switch (cmd) {
        case CLOSE_AV_TRANSMISSION_CMD:
            mMediaStreamStrategy->closeAVStreaming();
            break;
        case PAUSE_STREAMING:
            mMediaStreamStrategy->pauseStreaming();
            break;
        case RESUME_STREAMING: {
            ctrlRet = mMediaStreamStrategy->resumeStreaming();
        }
            break;
        case CLOSE_BIDIRECTION_INTERCOM:
            mMediaStreamStrategy->closeTwoWayInterCom();
            break;
        default:
            LOG808E("FATAL ERROR!!! Unknown control cmd %.2x", cmd);
            break;
    }
    return ctrlRet;
}

void adasplus::MediaStreamContext::switchBitStreamType(BYTE streamType) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! Channel[%d] The MediaStreamingStrategy is not setup for action %s",
                mLogicChannelNum, __FUNCTION__);
        return;
    }
    mMediaStreamStrategy->switchBitStreamType(streamType);
}

int adasplus::MediaStreamContext::startStreaming() {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! We need to setup StreamingStrategy before start streaming action");
        return -1;
    }
    return mMediaStreamStrategy->startStreaming();
}

void adasplus::MediaStreamContext::setupLiveVideoDataSource(
        std::shared_ptr<LiveVideoDataSource> liveVideoDataSource) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! The MediaStreamingStrategy is not setup for action %s", __FUNCTION__);
        return;
    }
    mMediaStreamStrategy->setupLiveVideoDataSource(liveVideoDataSource);
}

void adasplus::MediaStreamContext::setupLiveAudioDataSource(
        std::shared_ptr<LiveAudioDataSource> liveAudioDataSource) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! The MediaStreamingStrategy is not setup for action %s", __FUNCTION__);
        return;
    }
    mMediaStreamStrategy->setupLiveAudioDataSource(liveAudioDataSource);
}

void adasplus::MediaStreamContext::setPlatformTag(const char *tag) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! The MediaStreamingStrategy is not setup for action %s", __FUNCTION__);
        return;
    }
    mMediaStreamStrategy->setupPlatformTag(tag);
}
```
在上面的代码当中,会有一个潜在的问题,就是:

```cpp
void adasplus::MediaStreamContext::closeStreaming(adasplus::CloseStreamingCmd closeType) {
    if (mMediaStreamStrategy == nullptr) {
        LOG808E("FATAL STATE ERROR!!! Channel [%d] --> The MediaStreamingStrategy is not setup for action %s",
                mLogicChannelNum, __FUNCTION__);
        return;
    }
    switch (closeType) {
        case CLOSE_AUDIO_N_VIDEO:
            mMediaStreamStrategy->closeAVStreaming();
            break;
        case CLOSE_VIDEO_ONLY:
            mMediaStreamStrategy->closeVideoStreaming();
            break;
        case CLOSE_AUDIO_ONLY:
            mMediaStreamStrategy->closeAudioStreaming();
            break;
        default:
            LOG808E("FATAL ERROR!!! on channel [%d] --> Unknown close type %.2x", mLogicChannelNum, closeType);
            break;
    }
}
```

上面的代码当中,我们在执行`mMediaStreamStrategy->closeAudioStreaming();`时,`mMediaStreamStrategy`指向的对象很可能被`delete`掉,
因此最好的方式是使用`std::shared_ptr<IMeidaStreamStrategy>`来维护`mMediaStreamStrategy`.
但是有个问题,就是`IMediaStreamStrategy`本身是一个`虚基类`,无法直接被实例化,这样我们就无法使用`std::shared_ptr<>`了

--> 目前看来,只有`template`才能满足我们的需求 --> 
