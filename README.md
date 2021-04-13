# AutomaticPosting-koapy
> koapy 일부를 나한테 맞게 수정

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
