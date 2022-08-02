# Broadcast系统简述

### 1.简介

​	野狐围棋转播系统后端服务主要由生产、观战管理、转播三个部分构成，通过一定的逻辑将相关的棋局转播到各个客户端。

​	生产的数据来自于两个端，老PC转播（PCBroadcastAdaptor）和新框架对局（GameSvr），通过观战管理服（OBBroadcastSvrMgr）后转播到转播服（OBBroadcastSvr）对客户端提供服务。

### 2.数据生产

#### 2.1老PC转播（PCBroadcastAdaptor）

格式的魔鬼十多个节目是

#### 2.2新框架对局（GameSvr）

个服上东国际弄到手给第三方第三方神鼎飞丹砂发的什么发顺丰递四方速递收到

### 3.观战管理（OBBroadcastSvrMgr）

```c++
// Server在初始化时启动一个定时器来定时扫描获取需要转播的数据
int32_t Server::Init(){
    NTimer.StartTimer(NCfgData.scan_room_interval_ms, cxx::bind(&RoomManager::ScanGameSvrRoom, &m_room_mgr, _1));
}    
```

```c++
int32_t RoomManager::ScanGameSvrRoom(int64_t cb_timer_id) {

    if (!NDataGuard.IsMaster()) {
        return pebble::kTIMER_BE_CONTINUED;
    }

    if (IsScanning())
        return pebble::kTIMER_BE_CONTINUED;
    // 通过m_scanning字段控制是否正在扫描中
    SetScanningState(true);
    // 通过RPC获取GameSvr，同时开启协程获取RoomInfo
    GetGameSvrInfoFromGameSvrMgr();
    // 通过RPC获取room信息，并update
    GetPCBroadcastRoomInfo();

    return pebble::kTIMER_BE_CONTINUED;
}


int32_t RoomManager::GetGameSvrInfoFromGameSvrMgr(){
    ...
    NRPCGameSvrMgrBroadcast(-1).GetAllGameSvrInfo(req, [this](int32_t ret_code, const proto::broadcast::GetAllGameSvrInfoResponse& response){
        ...
        NPebbleServer.MakeCoroutine(cxx::bind(&RoomManager::GetRoomInfo, this, response));
    });
    return ret;
}


int32_t RoomManager::GetRoomInfo(const proto::broadcast::GetAllGameSvrInfoResponse& response){
		// 获取详细的room信息
	}


int32_t RoomManager::GetPCBroadcastRoomInfo() {
    proto::broadcast::GetAllRoomBriefInfoRequest get_room_info_req;

    NRPCPcBroadcast(-1).GetAllRoomBriefInfo(
        get_room_info_req, [this](int32_t ret_code, const proto::broadcast::GetAllRoomBriefInfoResponse& get_room_info_response) {
            ...
            UpdateRoomInfoByRpcResponse(get_room_info_response);
            // 对所有room排序（筛选在观战列表中的房间对其排序）
		    NRoomSortMgr.SortPCRoom();
     });
}
```

```c++
void RoomSortManager::SortPCRoom(){
	...
    int32_t modify_index = (this->m_app_current_index + 1) % 2;		
    ClearReserveRoomSort();
    NRoomMgr.ForEachRoom([this, modify_index](RoomPtr room){
        // 筛选观战列表
        if(!room->InWatchRoomList()){
            return;
        }			
		// 加入相应的观战列表
        if(room->IsMatch()){
            this->m_app_room_sort[modify_index].match_sorted_room_vec.push_back(room);

            this->m_app_room_sort[modify_index].all_sorted_room_vec.push_back(room);
            this->m_app_room_sort[modify_index].all_with_feidao_sorted_room_vec.push_back(room);	
        }
		...

    });
	// 更新排序
    m_app_room_sort[modify_index].Sort();
    m_app_current_index = modify_index;

}

// 至此，完成了对观战列表的维护，只需要等待客户端通过协议拉取数据即可
RegisterClientMsg(GET_EDU_ROOM_LIST_BY_TYPE, GetEduRoomListByType);
RegisterClientMsg(GET_HOME_ROOM_LIST, GetHomeRoomList);
RegisterClientMsg(GET_ROOM_LIST_BY_TYPE, GetRoomListByType);
```



### 4.转播（OBBroadcastSvr）

```c++
// 客户端通过 ENTER_ROOM = 100; GET_ROOM_INFO = 102; 获取全量操作序列

// OPERATION_NOTIFY = 103;	//棋局变化通知,由服务器主动通知给客户端  notify:OperationNotify
void Room::Broadcast2ClientOperationData(const proto::broadcast::Operation& opt){ //广播最新操作		
    proto::broadcast_svr::OperationNotify notify;
    notify.mutable_opt()->CopyFrom(opt);
    NYHProcessor.SendFGWClientsMsg(m_player_ids, proto::broadcast_svr::OPERATION_NOTIFY, &notify);
}
```

