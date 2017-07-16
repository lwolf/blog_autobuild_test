+++
author = "Sergey Nuzhdin"
categories = ["sql", "coding"]
date = "2014-06-17"
title= "Getting epoch date for highcharts from sqlalchemy"
type = "post"
aliases = [
    "/blog/2014/06/17/getting-epoch-date-for-highcharts-from-sqlalchemy/"
]
+++

Short sqlalchemy query to get highcharts ready data from database from datetime field.

    select count(1), extract(epoch from date_trunc('hour', created_at)) from bid group by extract(epoch from date_trunc('hour', created_at));
        db.session
            .query(
                extract('epoch', func.DATE(Bid.created_at)).label('dt'),
                func.count()
            )
            .filter(Bid.ownership == o)
            .filter(Bid.created_at > days_to_count )
            .group_by('dt')
            .order_by('dt')
        )
