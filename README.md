# AutomaticPosting-koapy
> koapy 일부를 나한테 맞게 수정
## 사용법
> 기본 사용법과 동일하나, 받은 데이터 처리 방식이 다름
```python
from koapy import KiwoomOpenApiPlusEntrypoint

with KiwoomOpenApiPlusEntrypoint() as context:
    # 로그인 처리
    context.EnsureConnected()

    # 이벤트를 스트림으로 반환하는 하위 함수 직접 사용 예시 (위의 상위 함수 내부에서 실제로 처리되는 내용에 해당)
    rqname = '업종별주가요청'
    # trcode = 'OPT10001'
    trcode = 'OPT20002'
    screenno = '0073'
    inputs = {'시장구분': '0', '업종코드': '013'}
    single = {}
    multi = []
    # 아래의 함수는 gRPC 서비스의 rpc 함수를 직접 호출함
    # 따라서 event 메시지의 구조는 koapy/backend/kiwoom_open_api_plus/grpc/KiwoomOpenApiPlusService.proto 파일 참조
    for event in context.TransactionCall(rqname, trcode, screenno, inputs):
        names = event.single_data.names
        values = event.single_data.values
        multi_names = event.multi_data.names
        multi_values = event.multi_data.values

        # multi_names 없는 경우, multi_names는 리스트의 index가 되기 때문에 multi_values를 순회하면서 single_names와 매칭 해준다.
        for value in multi_values:
            temp_dict = {}
            for n, v in zip(names, value.values):
                temp_dict[n] = v
            multi.append(temp_dict)
    # 전체 결과 출력 (싱글데이터)
    print('Got OPT2002 Tr Data (using TransactionCall):')
    print(single)
    print(multi)
    # 현재가 값만 출력
    # price = output['현재가']
    # print(price)
```
## 수정 코드
```python
# koapy/koapy/backend/kiwoom_open_api_plus/grpc/event/KiwoomOpenApiPlusEventHandlers.py

# koapy.backend.kiwoom_open_api.plus.event.KiwoomOpenApiPlusTrEventHandler.OnReceiveTrData()
def OnReceiveTrData(self, scrnno, rqname, trcode, recordname, prevnext, datalength, errorcode, message, splmmsg):
        if (rqname, trcode, scrnno) == (self._rqname, self._trcode, self._scrnno):
            response = KiwoomOpenApiPlusService_pb2.ListenResponse()
            response.name = 'OnReceiveTrData'  # pylint: disable=no-member
            response.arguments.add().string_value = scrnno  # pylint: disable=no-member
            response.arguments.add().string_value = rqname  # pylint: disable=no-member
            response.arguments.add().string_value = trcode  # pylint: disable=no-member
            response.arguments.add().string_value = recordname  # pylint: disable=no-member
            response.arguments.add().string_value = prevnext  # pylint: disable=no-member

            should_stop = prevnext in ['', '0']
            repeat_cnt = self.control.GetRepeatCnt(trcode, recordname)

            if len(self._single_names) > 0:
                values = [self.control.GetCommData(
                    trcode, recordname, 0, name).strip() for name in self._single_names]
                response.single_data.names.extend(
                    self._single_names)  # pylint: disable=no-member
                response.single_data.values.extend(
                    values)  # pylint: disable=no-member

            if repeat_cnt > 0:
                if len(self._multi_names) > 0:
                    rows = [[self.control.GetCommData(trcode, recordname, i, name).strip(
                    ) for name in self._multi_names] for i in range(repeat_cnt)]
                    response.multi_data.names.extend(
                        self._multi_names)  # pylint: disable=no-member
                    for row in rows:
                        if self._is_stop_condition(row):
                            should_stop = True
                            if self._include_equal:
                                response.multi_data.values.add().values.extend(row)  # pylint: disable=no-member
                            break
                        response.multi_data.values.add().values.extend(row)  # pylint: disable=no-member
                else:
                    # when not exist multi_names
                    print('repeat cnt: ', repeat_cnt)
                
                    rows = [[self.control.GetCommData(trcode, recordname, i, name).strip(
                    ) for name in self._single_names] for i in range(repeat_cnt)]
                    response.multi_data.names.extend(
                        map(str, range(repeat_cnt)))  # pylint: disable=no-member

                    for row in rows:
                        if self._is_stop_condition(row):
                            should_stop = True
                            if self._include_equal:
                                response.multi_data.values.add().values.extend(row)  # pylint: disable=no-member
                            break
                        response.multi_data.values.add().values.extend(row)  # pylint: disable=no-member

                    self.logger.warning(
                        'Repeat count greater than 0, but no multi data names available.')

            self.observer.on_next(response)

            if should_stop:
                self.observer.on_completed()
                return
            else:
                try:
                    KiwoomOpenApiPlusError.try_or_raise(
                        self.control.RateLimitedCommRqData(rqname, trcode, int(prevnext), scrnno, self._inputs))
                except KiwoomOpenApiPlusError as e:
                    self.observer.on_error(e)
                    return
```
